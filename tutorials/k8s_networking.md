# Kubernetes Networking

Kubernetes provides several mechanisms for network communication inside and outside the cluster.
In this tutorial we cover **Service types** and **Ingress**, and learn how Kubernetes integrates with AWS to provision cloud load balancers.

> [!NOTE]
> This tutorial assumes the YoloService and YoloFrontend Deployments are running in your cluster from the previous tutorials.
> If not, deploy them now before continuing.

## Service Types

By default, every Service you create is `ClusterIP`. Your existing `yolo-frontend-svc` is a `ClusterIP` service, which is why you had to use `port-forward` to reach it from outside the cluster.

## NodePort service type

A **NodePort** service opens a static port (in the range `30000-32767`) on **every node** in the cluster.
Any traffic arriving at `<node-ip>:<node-port>` is forwarded to the matching Pods.

Update the YoloFrontend service to type `NodePort`:

```yaml
# k8s/yolo-frontend-nodeport.yaml

apiVersion: v1
kind: Service
metadata:
  name: yolo-frontend-node-port-svc
spec:
  type: NodePort
  selector:
    app: yolo-frontend
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30080
```

```bash
kubectl apply -f k8s/yolo-frontend-nodeport.yaml
kubectl get svc yolo-frontend-node-port-svc
```


Open your browser at:

```
http://<worker-node-public-ip>:30080
```

> [!NOTE]
> Make sure the `kubeadm-cluster-node-sg` security group allows inbound TCP traffic on port `30080`.
> If not, add an inbound rule in the AWS EC2 console for your IP.


## LoadBalancer service type

A **LoadBalancer** service requests a cloud load balancer that distributes traffic to all healthy nodes automatically.

Update the service type to `LoadBalancer`:

```yaml
# k8s/yolo-frontend-lb.yaml

apiVersion: v1
kind: Service
metadata:
  name: yolo-frontend-lb-svc
spec:
  type: LoadBalancer
  selector:
    app: yolo-frontend
  ports:
    - port: 80
      targetPort: 3000
```

```bash
kubectl apply -f k8s/yolo-frontend-lb.yaml
kubectl get svc yolo-frontend-lb-svc -w
```

The `EXTERNAL-IP` column shows `<pending>` - and stays that way:

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
yolo-frontend-lb-svc   LoadBalancer   10.96.144.201   <pending>     80:31234/TCP   1m
```

**Why is it stuck?**

Kubernetes is cloud-agnostic. It knows *you want* an external load balancer, but it cannot create one in AWS by itself.
It relies on a **Cloud Controller Manager (CCM)** to communicate with the AWS API.
Your cluster does not have one installed yet - that is the next step.

## The AWS Cloud Controller Manager

The **Cloud Controller Manager (CCM)** is a Kubernetes add-on that bridges the Kubernetes API with your cloud provider.
For AWS, the CCM:

- **Provisions load balancers** - watches for `LoadBalancer` Services and creates/updates/deletes actual AWS ELBs.
- **Manages node lifecycle** - removes cluster nodes when their EC2 instance is terminated.
- **Labels nodes** with their AWS Availability Zone and instance type.


### Tag your EC2 instances and subnets

The CCM identifies which EC2 instances belong to your cluster using a tag.
In the AWS console, add this tag to **both** your control-plane and worker instances:

| Key | Value |
|-----|-------|
| `kubernetes.io/cluster/<your-name>` | `owned` |

1. Go to the EC2 console and select your control-plane instance.
2. Click the **Tags** tab and then **Add/Edit Tags**.
3. Add the tag with key `kubernetes.io/cluster/<your-name>` and value `owned`.
4. Repeat the same for your worker node.


Last thing, the CCM needs the `providerID` field to be set on each node object in Kubernetes.
This field is used to link the Kubernetes node object to the corresponding EC2 instance in AWS.
If you created your cluster using `kubeadm`, this field is not populated by default.
To set it, run the following command for each node in your cluster.

Run the following script from the control plane. It uses the AWS CLI to look up each node's instance ID and AZ by its private IP:

```bash
rm -f /usr/local/bin/aws
hash -r
sudo snap install aws-cli --classic

for NODE in $(kubectl get nodes --no-headers -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address | awk '{print $1}'); do
  IP=$(kubectl get node $NODE -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
  INSTANCE=$(aws ec2 describe-instances \
    --filters "Name=private-ip-address,Values=$IP" \
    --query "Reservations[0].Instances[0].InstanceId" --output text)
  AZ=$(aws ec2 describe-instances \
    --filters "Name=private-ip-address,Values=$IP" \
    --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)
  kubectl patch node $NODE -p "{\"spec\":{\"providerID\":\"aws:///$AZ/$INSTANCE\"}}"
done
```



### Install the AWS CCM

The nodes already carry the `kubeadm-cluster-node-role` IAM role, which grants the CCM the AWS permissions it needs (ELB management, EC2 describe calls, etc.).

Add the Helm repository:

```bash
helm repo add aws-cloud-controller-manager https://kubernetes.github.io/cloud-provider-aws
helm repo update
```

Create a values file - replace `<your-name>` with your actual cluster name:

```yaml
# k8s/ccm-values.yaml

args:
  - --v=2
  - --cloud-provider=aws
  - --cluster-name=<your-name>
  - --configure-cloud-routes=false
```


Install the CCM into the `kube-system` namespace:

```bash
helm install aws-cloud-controller-manager \
  aws-cloud-controller-manager/aws-cloud-controller-manager \
  --namespace kube-system \
  -f k8s/ccm-values.yaml
```

Verify the CCM pod is running:

```bash
kubectl get pods -n kube-system -l k8s-app=aws-cloud-controller-manager
```


With the CCM running, it will detect the pending `LoadBalancer` service and contact the AWS API to create an ELB:

```bash
kubectl get svc yolo-frontend-lb-svc -w
```

After a minute or two the `EXTERNAL-IP` column is populated with the ELB DNS hostname:

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP                                         PORT(S)       AGE
yolo-frontend-lb-svc   LoadBalancer   10.96.144.201   a1b2c3d4.eu-central-1.elb.amazonaws.com             80:31234/TCP  3m
```

Open `http://<elb-hostname>` in your browser to access the YoloFrontend through the load balancer.

![][k8s_networking_lb_service]


### The problem with one load balancer per service

Each `LoadBalancer` service provisions a **separate** ELB in AWS. For a cluster with many externally-accessible services this quickly becomes expensive and inflexible in temrs of routing and TLS configuration.

The industry solution is **Ingress**.

## Ingress

An **Ingress** is a Kubernetes object that defines HTTP/HTTPS routing rules.
A single **Ingress Controller** - backed by one shared LoadBalancer - handles all external HTTP traffic and forwards it to the correct Service based on hostname or URL path.

```
Internet ──► Load Balancer (AWS)
                └──► Ingress Controller Pod (Nginx)
                        ├── / ──────────────► yolo-frontend-svc:3000
                        └── /api ───────────► yolo-svc:8080
```

### Install the Nginx Ingress Controller via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Create a values file to specify the Load balancer type we need for AWS (Network Load Balancer):

```yaml
# k8s/ingress-nginx-values.yaml

controller:
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

Install the chart:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f k8s/ingress-nginx-values.yaml
```


### Expose YoloFrontend via Ingress

You have to know the ELB hostname to set up the Ingress rules, so wait until the `ingress-nginx-controller` service has an external IP:

```bash
kubectl get svc -n ingress-nginx
```

Copy the value in the `EXTERNAL-IP` column (e.g. `a1b2c3d4.eu-central-1.elb.amazonaws.com`).


Create an Ingress resource that routes all traffic to the frontend:

```yaml
# k8s/yolo-frontend-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yolo-frontend-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: <your-load-balancer-hostname>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yolo-frontend-svc
                port:
                  number: 3000
```

```bash
kubectl apply -f k8s/yolo-frontend-ingress.yaml
kubectl get ingress
```

Open `http://<ingress-nginx-elb-hostname>` - traffic flows through the single shared ELB, through the Nginx Ingress Controller, and into the YoloFrontend.

> [!TIP]
> `kubectl delete validatingwebhookconfiguration ingress-nginx-admission` if you want to be able to create Ingress resources without the `ingressClassName` field (i.e. default to nginx).


### How Ingress routing works

- **`ingressClassName: nginx`** - tells Kubernetes which Ingress Controller handles this rule. Multiple controllers can coexist in the same cluster; each one only processes Ingress objects that reference its own class.
- **`rules`** - each rule maps an optional hostname and/or URL path to a backend Service. You can add multiple rules to route different paths or domains to different services without provisioning additional load balancers.

# Exercises 


## :pencil2: (optional) Point a custom domain to your ELB via Route 53

In this exercise you will map a personal subdomain (e.g. `john.exit-zero.click`) to the ELB that was created for the Ingress Controller.

1. **Get the ELB hostname**

   ```bash
   kubectl get svc -n ingress-nginx
   ```

   Copy the value in the `EXTERNAL-IP` column (e.g. `a1b2c3d4.eu-central-1.elb.amazonaws.com`).

2. Open Route 53 in the AWS console
3. Navigate to **Services → Route 53 → Hosted zones** and click on `exit-zero.click`.
4. Create a new DNS record set:

   Click **Create record** and fill in the following:

   | Field | Value |
   |-------|-------|
   | Record name | `<your-name>` (e.g. `john`) |
   | Record type | `CNAME` |
   | Value | The ELB hostname from step 1 |
   | TTL | `300` |

   Click **Create records**.

5. Navigate to `http://john.exit-zero.click` in your browser.
   Traffic flows: DNS → ELB → Nginx Ingress Controller → YoloFrontend.


[k8s_services]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_services.png



[k8s_networking_lb_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_lb_service.png
[k8s_networking_nginx_ic]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_nginx_ic.png
