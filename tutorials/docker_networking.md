# Docker Networking

## Docker network sandbox and drivers

The core idea of containers is **isolation**, so how can a container communicate with other container over the network? 
How can a container communicate with the public internet? 
It must be using the host machine network resources. 
How can that be implemented while keeping the host machine secure enough? 

Docker implement a virtualized layer for container networking, enables communication and connectivity between Docker containers, as well as between containers and the external network.
It includes all the traditional stack we know - unique IP address for each container, virtual network interface that the container sees, default gateway and route table. 

Below is the virtualized model scheme. In Docker, this model is implemented by [`libnetworking`](https://github.com/moby/libnetwork) library: 

![][docker_sandbox]

A **sandbox** is an isolated network stack. It includes Ethernet interfaces, ports, routing tables, and DNS configurations.

**Network Interfaces** are virtual network interfaces (E.g. `veth` ).
Like normal network interfaces, they're responsible for making connections between the container and the rest of the world. 

Network interfaces are connected the sandbox to networks.

A **Network** is a group of network interfaces that are able to communicate with each-other directly. 
An implementation of a Network could be a Linux bridge, a VLAN, etc. 

This networking architecture is not exclusive to Docker. Docker is based on an open-source pluggable architecture called the [**Container Network Model** (CNM)](https://github.com/moby/libnetwork/blob/master/docs/design.md). 

The networks that containers are connecting to are pluggable, using **network drivers**.
This means that a given container can communicate to different kind of networks, depending on the driver. 
Here are a few common network drivers docker supports:

- [`bridge`](https://docs.docker.com/network/bridge/): This network driver connects containers running on the **same** host machine. If you don't specify a driver, this is the default network driver.

- [`host`](https://docs.docker.com/network/host/): This network driver connects the containers to the host machine network - there is no isolation between the
  container and the host machine, and use the host's networking directly.

- [`overlay`](https://docs.docker.com/network/overlay/): Overlay networks connect multiple container on **different machines**,
  as if they are running on the same machine and can talk locally. 

- [`none`](https://docs.docker.com/network/none/): This driver disables the networking functionality in a container.

## The Bridge network driver

The `bridge` network driver is the default network driver used by Docker.
It creates an internal network bridge on the host machine and assigns a unique IP address to each container connected to that bridge.

Containers connected to the `bridge` network driver can communicate with each other using these assigned IP addresses.

A container receives an IP address out of the IP pool of the network it attaches to. 
The Docker daemon effectively acts as a DHCP server for each container.
Each network also has a **default subnet** mask and **gateway**.

In the same way, a container's hostname defaults to be the container's ID or name in Docker. 
You can override the hostname using `--hostname`.

## DNS services

Containers that are connected to the default bridge network inherit the DNS settings of the host, as defined in the `/etc/resolv.conf` configuration file in the host machine (they receive a copy of this file). 

Containers that attach to a custom network use Docker's embedded DNS server. 
The embedded DNS server forwards external DNS lookups to the DNS servers configured on the host machine.


# Containers storage

By default, all files created inside a container are stored on a writable container layer.
This means that the data doesn't persist when that container no longer exists.

Docker has two options for containers to store files on the host machine, so that the files are persisted even after the container stops: **volumes**, and **bind mounts**.

## Bind mounts 

**Bind mounts** provide a way to mount a directory or file **from the host machine into a container**.
Bind mounts directly map a directory or file on the host machine to a directory in the container. 

Let's take the `alonithuji/yolo-service:0.0.1` image as a concrete example.
The YoloService stores every uploaded image in `/app/uploads/` and records every prediction in an SQLite database at `/app/predictions.db` inside the container.
By default those files vanish when the container is removed - unless we bind-mount host paths in their place:

```bash
cd YoloService
docker run --rm -p 8080:8080 \
  -v $(pwd)/predictions.db:/app/predictions.db \
  -v $(pwd)/uploads:/app/uploads/ \
  --name yolo alonithuji/yolo-service:0.0.1
```

In this example, the two `-v` flags specify the bind mounts:
- `-v $(pwd)/uploads:/app/uploads/` maps the `uploads/` directory in your current working directory to `/app/uploads/` inside the container, so every uploaded image is stored on the host.
- `-v $(pwd)/predictions.db:/app/predictions.db` maps a single file on the host to `/app/predictions.db` inside the container, so every prediction record written to the SQLite database is persisted on the host.

Whenever the `yolo` container writes to either path, the data is actually stored on the host machine.
Every change the container makes is immediately reflected on the host, and vice-versa.

With these bind mounts in place, both the uploaded images and the predictions database survive even after the container is stopped or removed - they remain on your host machine.

Bind mounts are commonly used for development workflows, where file changes on the host are immediately reflected in the container without the need to rebuild or restart the container.
They also allow for easy access to files on the host machine, making it convenient to provide configuration files, logs, or other resources to the container.

## Volumes 

**Docker volumes** is another way to persist data in containers. 
While bind mounts are dependent on the directory structure and OS of the host machine, volumes are logical space that completely managed by Docker.
Volumes offer a higher level of abstraction, allow us to work with different kind of storages, e.g. volumes stored on remote hosts or cloud providers.
Volumes can be shared among multiple containers.

### Create and manage volumes

Unlike a bind mount, you can create and manage volumes outside the scope of any container.

```bash 
docker volume create my-vol
```

Let's inspect the created volume:

```bash
$ docker volume inspect my-vol
[
    {
        "CreatedAt": "2023-05-10T14:28:15+03:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

Inspecting the volume discovers the real location that the data will be stored on the host machine: `/var/lib/docker/volumes/my-vol/_data`.

The `local` volume driver is the default built-in driver that stores data on the local host machine.
But docker offers [many more](https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins) drivers that allow you to create different types of volumes that can be mapped to your container.
For example, the [Azure File Storage plugin](https://github.com/Azure/azurefile-dockervolumedriver) lets you mount Microsoft Azure File Storage shares to Docker containers as volumes.

# Exercises

### :pencil2: Persist Grafana and Prometheus using Volumes 

... 


### :pencil2: Docker networks and volumes


The [default bridge network](https://docs.docker.com/network/network-tutorial-standalone/#use-the-default-bridge-network) official tutorial demonstrates how to use the default bridge network that Docker sets up for you automatically. 

The [user-defined bridge networks](https://docs.docker.com/network/network-tutorial-standalone/#use-user-defined-bridge-networks) official tutorial shows how to create and use your own custom bridge networks, to connect containers running on the same host machine.




### :pencil2: Mount the Nginx conf file 

1. Run your NetflixMovieCatalog container.
2. Create an Nginx container which routes the traffic to your NetflixMovieCatalog container. 
   For simplicity, you don't need to wrap the Flask app with uWSGI. Instead, your Nginx communicates directly with Flask using a simple `proxy_pass` directive over HTTP protocol:
   ```text
   server {
     listen 80;
     server_name  localhost;
     location / {
       proxy_pass http://<netflix_movie_catalog>:8080;
      }
   } 
   ```
   
   While `<netflix_movie_catalog>` is the container IP of your NetflixMovieCatalog container.
3. Persist the `.conf` file of Nginx so you can kill the container and start a fresh one, and still the Nginx custom server configuration would be applied. 
4. Visit the app via Nginx. 

### :pencil2: Persist InfluxDB database

Your goal is to run InfluxDB container from the [previous exercise](docker_containers.md#pencil2-availability-test-system) while the data is stored persistently.

Make sure the data is persistent by `docker kill` the container and create a new container one.  

### :pencil2: Understanding user file ownership in docker

When running a container, the default user inside the container is often set to the `root` user, and this user has full control of the container's processes.
Since containers are isolated process in general, we don't really care that `root` is the user operating within the container.

But, when mounting directories from the host machine using the `-v` command, it is important to be cautious when using the root user in a container. Why?

We will investigate this case in this exercise...

1. On your host machine, create a directory under `~/test_docker`.
2. Run the `ubuntu` container while mounting `/test` within the container, into `~/test_docker` in the host machine:

```bash
docker run -it -v ~/test_docker:/test ubuntu /bin/bash
```

3. From your host machine, create a file within `~/test_docker` directory.
4. From the `ubuntu` container, list the mounted directory (`/test`), can you see the file you've created from your host machine?
   Who are the UID (user ID) and GID (group ID) owning the file?
5. From within the container, create a file within `/test`.
6. List `~/test_docker` from your host machine. Who are user and group owning the file created from the container?
7. Try to indicate the potential vulnerability: "If an attacker gains control of the container, they may be able to..."
8. Repeat the above scenario, but instead of using `-v`, use docker volumes. Starts by creating a new volume by: `docker create volume testvol`. Describe how using docker managed volume can reduce the potential risk.


### :pencil2: Investigate volume mounting

Use the `ubuntu` container to experiment with volume mounting and answer the following questions:

1. What happens when you mount an **empty docker volume** into a directory in the container in which files or directories exist?
2. What happens when you start a container and specify a docker volume name that does not exist?
3. What happens when you mount a path using **bind mount** of non-empty directory in the container?
4. What happens when you start a container and specify paths (both in the host machine and the container) that does not exist?

[docker_volumes]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_volumes.png


# Exercises

### :pencil2: The Packet's Journey: Container → Internet



### :pencil2: Inspecting container networking

Run the `busybox` image by:

```bash
docker run -it busybox /bin/sh
```

1. On which network this container is running?
2. What is the name of the network interface that the container is connected to, as it is seen from the host machine?
3. What is the name of the network interface that the container is connected to as it is seen from within the container?
4. What is the IP address of the container?
5. Using the `route` command, what is the default gateway ip that the container use to access the internet?
6. Provide an evidence that the container's default gateway IP is the IP address of the default bridge network on the host machine the container is running on.
7. What are the IP address(es) of the DNS server the container used to resolve hostnames? Provide an evidence that they are identical to the DNS servers host machine.
8. Create a new bridge network, connect your running container to this network.
9. Provide an evidence that the container has been connected successfully to the created network.
10. From the host machine, try to `ping` the container using both its IP addresses.
11. After you've connected the container to a custom bridge network, what are the IP address of the DNS server the container used to resolve hostnames? What does it mean?


### :pencil2: The Host network driver 

The Host network driver is a network mode in Docker where a container shares the network stack of the host machine.
When a container is run with the Host network driver, it bypasses Docker's virtual networking infrastructure and directly uses the network interfaces of the host.
This allows the container to have unrestricted access to the host's network interfaces, including all network ports. However, it also means that the container's network stack is not isolated from the host, which can introduce security risks.

Complete Docker's short tutorial that demonstrates the use of the host network driver:      
https://docs.docker.com/network/network-tutorial-host/



### :pencil2: Flask, Nginx, MongoDB

Your goal is to build the following architecture:

![](../.img/docker_nginx-flask-mongo.png)

- The Dockerfiles of the nginx and the flask apps can be found in our shared repo under `nginx_flask_mongodb`.
- The mongo app should be run using the pre-built [official Mongo image](https://hub.docker.com/_/mongo).
- The nginx and flask app should be connected to a custom bridge network called `public-net-1` network.
- In addition, the flask app the mongo should be connected to a custom bridge network called `private-net-1` network.
- The nginx should talk with flask using the `flask-app` hostname.
- The flask app should talk to the mongo using the `mongo` hostname.

### :pencil2: Overlay network using Calico






[docker_sandbox]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_sandbox.png
[docker_cache]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_cache.png
[docker_nginx_frontend_catalog]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_nginx_frontend_catalog.png