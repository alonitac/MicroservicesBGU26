
## Horizontal Pod Autoscaling

> [!NOTE]
> Before starting, make sure the `metrics-server` is installed in your cluster. 
> 
> - If working on Minikube: `minikube addons enable metrics-server`.
> - If working on another k8s cluster, [read here](https://github.com/kubernetes-sigs/metrics-server?tab=readme-ov-file#installation). 

A **HorizontalPodAutoscaler** (HPA for short) automatically updates the number of Pods to match demand, usually in a Deployment or StatefulSet.

The HorizontalPodAutoscaler controller periodically (by default every 15 seconds) adjusts the desired scale of its target (for example, a Deployment) to match observed metrics such as average CPU utilization, average memory utilization, or any other custom metric you specify.
The common use for HPA is to configure it to fetch metrics from a [Metrics Server](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-server) API (should be installed as an add-on). 
Every 15 seconds, the HPA controller queries the Metric Server and fetches resource utilization metrics (e.g. CPU, Memory) for each Pod that are part of the HPA. 
The controller then calculates the desired replicas, and scale in/out based on the current replicas.

Let's create an autoscaler object for the `nginx` Deployment:

```yaml
# k8s/hpa-autoscaler-demo.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

When defining the pod specification, the resource **Requests**, like CPU and memory must be specified.
This is used to determine the resource utilization and used by the HPA controller to scale the target up or down.
In the above example, the HPA will scale up once the Pod is reaching 50% of the `.resources.requests.cpu`. 

Next, let's see how the autoscaler reacts to increased load. 
To do this, you'll start a different Pod to act as a client. 
The container within the client Pod runs in an infinite loop, sending queries to the `nginx-service` service.

```bash 
kubectl run -it load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.001; do (wget -q -O- http://nginx-service &); done"
```