# Containers 

## Running your first container

Run the [`nginx`](https://hub.docker.com/_/nginx) container by: 

```bash 
docker pull nginx
docker run nginx
```

When you run this command, the following happens (assuming you are using the default DockerHub registry configuration):

1. Docker pulls the `nginx` image from DockerHub. 
2. Docker creates a new container from the `nginx` image. The `nginx` image is a ready-to-run container image that encapsulates the [NGINX web server](https://www.nginx.com/resources/glossary/nginx/) software, along with its dependencies and configuration. When the image is used to create a container, it provides a fully functional NGINX server environment, without the need to install and configure nginx on your machine.
3. Docker allocates a dedicated read-write filesystem to the container (which is completely different and isolated from the host machine fs). This allows a running container to create or modify files and directories in its local filesystem.
4. Docker creates a **virtual network interface** to connect the container to the network. This includes assigning an IP address to the container. By default, containers can connect to external networks using the host machine's network connection.
5. Docker starts the container.
6. When you type `CTRL+c` the container stops but is not removed. You can start it again or remove it. When a container is removed, its file system is deleted. 

## Container management and lifecycle

To see your **running** containers, type:

```bash
docker ps 
```

or add  `-a` flag to list also stopped containers:

```console
$ docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS                      PORTS     NAMES
d841a2fe07f9   nginx     "/docker-entrypoint..."   About a minute ago   Exited (0) 14 seconds ago             funny_blackburn
```

In the above output: 

- `d841a2fe07f9` is the **container ID** - a unique identifier assigned to each running container in Docker.
- `/docker-entrypoint...` is the (beginning) of the actual linux command that has run to initiate the process of the container. 
- `funny_blackburn` is a random alphabetical name that docker assigned to the container. 



### Published ports

By default, when you run a container using the `docker run` command, the container doesn't expose any of its ports to the outside world.
To make a port available to services outside of Docker, or to Docker containers running on a different network, use the `--publish` or `-p` flag. 
This creates a firewall rule in the container, mapping a container port to a port on the host machine to the outside world.

Here's an example:

```bash
docker run --name nginx3 -p 8080:80 nginx
```

`-p 8080:80` maps port 80 **in the container** to port 8080 **in the host machine**. 

You can then access the nginx web server by opening a web browser and navigating to `http://localhost:8080`.

More docker networking topics will be covered in next chapters. 

Explore the running nginx container logs, can you see a log indicating your request you've just performed from the web browser? 
Try to run the container without the `-p` flag and check that the nginx container is not accessible.  


### Set environment variables for containers

When running a container using the `docker run` command, you can specify environment variables using the `-e` or `--env` flag.
For example:

```bash
docker run -d -e MY_VAR=my_value --name nginx4 nginx
```

### Override the default command 

When the nginx image was built (we will build our own images soon), Docker allows us to specify a default command that defines what is executed when a container is started from the image.
However, you can override the default execution command by providing a new command as arguments when running the `docker run` command.

To override the default command, simply append the desired command to the **end** of the `docker run` command.

Let's say you want to run the same above `nginx` container, but you want to modify the default command so nginx is running in debug mode. You can override the default command by:

```bash 
docker run nginx nginx-debug -g 'daemon off;'
```

In the above example, the `nginx` container will be initiated using the command `nginx-debug -g 'daemon off;'`. 

Here is another very useful example: 

```bash 
docker run -it nginx /bin/bash
```

The command starts a new Docker container and launches an interactive terminal session within the container.

Here's what each part of the command does:

1. `docker run` instructs Docker to create and start a new container.
2. `-it` is a combination of two flags: `-i` keeps STDIN open for the container, and `-t` allocates a pseudo-TTY to allow interaction with the container's terminal.
3. `nginx` refers to the Nginx image.
4. `/bin/bash` specifies the command to be executed within the container, in this case, launching a Bash shell.

When you run this command, a new container based on the Nginx image is created, and you are provided with an interactive, fresh and beloved `bash` terminal session inside the container.
This allows you to directly interact with the Nginx container's files, run commands, and perform operations within the isolated containerized environment.

Feel free to go wild, you are within a container :-)

### Bind mounts 

**Bind mounts** provide a way to mount a directory or file **from the host machine into a container**.
Bind mounts directly map a directory or file on the host machine to a directory in the container. 

For example:

```bash
docker run --rm -p 8080:80 -v /path/on/host:/etc/nginx/conf.d/ --name my-nginx nginx:latest
```

In this example, we're running an Nginx container while the `-v /path/on/host:/etc/nginx/conf.d/` flag specifies the bind mount.
It maps the directory `/path/on/host` on the host machine to the `/etc/nginx/conf.d/` directory inside the container.

Whenever the `my-nginx` container is accessing `/etc/nginx/conf.d/` path, the directory it actually sees is `/path/on/host` on the host machine.
More than that, every change the container does to the `/etc/nginx/conf.d/` directory will reflect in the corresponding location on the host machine, `/path/on/host`, due to the bind mount.
The other way also applies, changes made on the host machine in `/path/on/host` will be visible inside the container under `/etc/nginx/conf.d/`.

### Interacting with containers

In addition to starting a new container with an interactive terminal session using docker run `-it`, you can also interact with **running** containers using the `docker exec` command.

The `docker exec` command allows you to execute a command inside a running container. Here's the basic syntax:

```bash 
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

Let's see it in action...

Start a new `nginx` container and keep it running. Give it a meaningful name instead the one Docker generates: 

```bash 
docker run --name my-nginx nginx 
```

Make sure the container is up and running. Since the running container occupying your current terminal session, open up another terminal session and execute:

```console
$ docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS     NAMES
89cf04f27c04   nginx     "/docker-entrypoint.…"   About a minute ago   Up About a minute   80/tcp    my-nginx
```

Now say we want to debug the running `nginx` container, and perform some maintenance tasks, or executing specific commands within the containerized environment, we can achieve it by:

```bash 
docker exec -it my-nginx /bin/bash
```

And you're in... You can execute any command you want within the running `my-nginx` container. 

**Tip**: if you don't know the container name, you can `exec` a command also using the container id:

```bash 
docker exec -it 89cf04f27c04 /bin/bash
```

### Inspecting a container 

The `docker inspect` command is used to retrieve detailed information about Docker objects such as containers, images, networks, and volumes. It provides a comprehensive JSON representation of the specified object, including its configuration, network settings, mounted volumes, and more.

The basic syntax for the docker inspect command is:

```bash 
docker inspect [OPTIONS] OBJECT
```

Where `OBJECT` represents the name or ID of the Docker object you want to inspect.


Inspect your running container by:

```console
$ docker inspect my-nginx
....
```

# Exercises

### :pencil2: Deploy a containerized Nginx 

In this exercise, you will deploy a containerized Nginx web server in your EC2 instance using Docker and GitHub Actions.

> [!IMPORTANT]
> Before starting, **you must remove the existing installation of Nginx** from your instance (or create a new instance alternatively).  

1. In your PolybotInfra repo, from branch `main`, create a feature branch called `feature/deploy_nginx`.
2. Update your deployment script (or create one if needed) to deploy a containerized Nginx web server in your EC2.
   You can use the following command to run the Nginx container:

   ```bash
   docker stop mynginx || true # gracefully stop the container if it is already running
   docker rm mynginx || true # remove the container if it is already running
   docker run -d --name mynginx -p 443:443 -v /home/ubuntu/conf.d:/etc/nginx/conf.d/  nginx # run the container in detached mode (-d)
   ```
3. Update your CI/CD workflow in GitHub Actions to deploy the Nginx container in your EC2 instance.
   - You might want to copy your `.conf` files from the repo source code into the `/home/ubuntu/conf.d` directory in your EC2 instance.
   - You might want to copy/generate if not exists the HTTPS certificates (feel free to add `-v` in the `docker run` command to mount the certificates directory from the host machine into the container).
4. Commit and push your changes to the `feature/deploy_nginx` branch.
5. Merge into `dev`, let the CI/CD workflow run and deploy the Nginx container in your EC2 instance. Test your dev bot availability. 
6. Create a PR (Pull Request) from `feature/deploy_nginx` into `main`.
7. Once the PR is approved and merged, the Nginx container should be deployed in your EC2 instance.


### :pencil2: Deploy a containerized Prometheus and Grafana

In this exercise, you will deploy a containerized Prometheus and Grafana in your monitoring EC2 instance using Docker.

> [!NOTE]
> It's not a mandatory to automate the deployment of your monitoring stack (using GitHub Actions workflows). 
> 
> If you decide to do so, it's recommended to work directly on your instance, and once things work, integrate a GitHub Actions workflow to automate the deployment.

- Both [Prometheus](https://hub.docker.com/r/prom/prometheus) and [Grafana](https://hub.docker.com/r/grafana/grafana) are available as official Docker images on DockerHub.
- Similarly to what done in Nginx, you should create a small deployment script to deploy the Prometheus and Grafana containers in your monitoring EC2 instance.
- Grafana is a visualization tool that can be used to visualize the metrics collected by Prometheus.
  Hence, the Grafana container should talk and fetch metrics from the Prometheus container.
  
  Here's how to do so:
  - Once you have the Grafana container running, you can access its web UI by navigating to `http://<monitoring-ec2-ip>:3000` in your web browser
  - The default username and password are both `admin`.
  - Set up Prometheus data source in Grafana:
    - Log into Grafana.
    - On the left panel, click **Connections** → **Data sources** → **Add data source**.
    - Select **Prometheus** and enter the URL: `http://<prometheus-container-ip>:9090`.
    - Click **Save & Test**.
  
  **Advanced options**: If you don't want to work with containers ip, you can use `docker network` to create a custom network for Grafana and Prometheus containers, then connect the containers to the network and use **the container name** (gave by the `--name` flag) as a stable DNS name (read more how to do so).
  
  - On the left panel enter the **Explore** panel and try to get a graph of availability metrics over time. 
- Make sure you're able to see Prometheus' metrics in the Grafana **Explore** page. 



[docker_grafana-ts]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_grafana-ts.png
[NetflixMovieCatalog]: https://github.com/exit-zero-academy/NetflixMovieCatalog
