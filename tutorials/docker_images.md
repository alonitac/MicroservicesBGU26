# Docker Images - Build, Tag, and Push

So far you've been running containers from images that someone else built and pushed to DockerHub.
Now it's time to build your own.

You'll take the [YoloService](https://github.com/alonitac/YoloService) source code and package it into a Docker image yourself - the same image you've been using all along.

## Building an image

To build an image out of the YoloService source code, run this command *in the repository directory (`YoloService`)*:

```bash
docker build -t yolo-service:0.0.1 .
```

- `docker build` reads a file called `Dockerfile` and executes each instruction in order.
- `-t yolo-service:0.0.1` assigns the name `yolo-service` and the tag `0.0.1` to the resulting image.
- `.` is the **build context** - the directory whose files are made available to the build.

Watch the output as Docker works through each step. The first build will take a few minutes - it needs to download the base image and install all Python dependencies.

When it finishes, verify the image is available locally:

```bash
docker images
```

```console
REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
yolo-service   0.0.1     3a8f2c1d4e01   10 seconds ago   1.02GB
```

Now let's look at what's inside that `Dockerfile` and understand what Docker actually did.

## The Dockerfile

A **Dockerfile** is a plain text file that contains a sequence of instructions telling Docker how to build an image.
Each instruction adds a layer on top of the previous one (more on layers shortly).

Here is the Dockerfile from the YoloService repository:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install -r torch-requirements.txt
RUN pip install -r requirements.txt

EXPOSE 8080

CMD ["python", "app.py"]
```

Let's walk through each instruction:

| Instruction | What it does |
|---|---|
| `FROM python:3.11-slim` | Starts from an official Python 3.11 base image. The `slim` variant is a minimal Debian-based image - smaller than the full image, without unnecessary tools. Every image must begin with `FROM`. |
| `WORKDIR /app` | Sets `/app` as the working directory for all subsequent instructions. If the directory doesn't exist, Docker creates it. Commands like `RUN`, `COPY`, and `CMD` all operate relative to this path. |
| `COPY . .` | Copies everything from the **build context** (the directory you run `docker build` from) into `/app` inside the image. The first `.` is the source (host), the second `.` is the destination (image). |
| `RUN pip install -r torch-requirements.txt` | Executes a shell command during the build. Installs the PyTorch dependencies. This happens at build time, not at runtime. |
| `RUN pip install -r requirements.txt` | Installs the remaining Python dependencies. |
| `EXPOSE 8080` | Documents that the container listens on port 8080. This is informational only - it doesn't actually publish the port. You still need `-p` when running the container. |
| `CMD ["python", "app.py"]` | Defines the default command to run when a container starts from this image. Written in **exec form** (a JSON array), which is preferred because it doesn't invoke a shell and handles signals correctly. |

## Docker Image Layers

Every instruction in a Dockerfile that modifies the filesystem (`FROM`, `COPY`, `RUN`) creates a new **layer**.
Think of layers as a stack of filesystem snapshots:

```
┌─────────────────────────────────┐
│  CMD ["python", "app.py"]       │  ← Not a layer (metadata only)
├─────────────────────────────────┤
│  RUN pip install requirements   │  ← layer 5
├─────────────────────────────────┤
│  RUN pip install torch-req      │  ← layer 4
├─────────────────────────────────┤
│  COPY . .                       │  ← layer 3
├─────────────────────────────────┤
│  WORKDIR /app                   │  ← layer 2
├─────────────────────────────────┤
│  FROM python:3.11-slim          │  ← layer 1 (base image)
└─────────────────────────────────┘
```

Each layer only stores the **diff** - the files added or changed by that instruction relative to the layer below it.
The final image is the union of all layers stacked together.

You can inspect the layers of any image with:

```bash
docker inspect alonithuji/yolo-service:0.0.1
```

## Layer Caching

Docker is smart: when you rebuild an image, it doesn't re-execute every instruction from scratch.
Instead, it checks whether anything has changed since the last build. If a layer's instruction and its inputs are identical to a previous build, Docker **reuses the cached layer** and skips the step entirely.

This is called the **layer cache**, and it can make rebuilds dramatically faster.

**The catch**: once a layer is invalidated (because something changed), **all subsequent layers are also invalidated** and must be rebuilt - even if they haven't changed themselves.

### Caching in action

Clone the YoloService repository and build the image:

```bash
docker build -t yolo-service:my-build .
```

Watch the output - the first build downloads and installs everything from scratch. It will take a few minutes.

Now build again immediately, without changing anything:

```bash
docker build -t yolo-service:my-build .
```

```console
 => CACHED [2/6] WORKDIR /app                                              0.0s
 => CACHED [3/6] COPY . .                                                  0.0s
 => CACHED [4/6] RUN pip install -r torch-requirements.txt                 0.0s
 => CACHED [5/6] RUN pip install -r requirements.txt                       0.0s
```

Every step is served from cache. The rebuild completes in under a second.

Now make a small change to the application code - edit `app.py` and add a comment to the top:

```bash
echo "# my change" >> app.py
docker build -t yolo-service:my-build .
```

```console
 => CACHED [2/6] WORKDIR /app                                              0.0s
 => [3/6] COPY . .                                                         0.3s  ← cache miss!
 => [4/6] RUN pip install -r torch-requirements.txt                        ...   ← re-runs
 => [5/6] RUN pip install -r requirements.txt                              ...   ← re-runs
```

Layer 3 (`COPY . .`) detects that the source files changed, so its cache is invalidated.
Because layers 4 and 5 come *after* it in the stack, they are also invalidated and pip reinstalls everything - even though `requirements.txt` itself didn't change at all.

This is the core problem with the original Dockerfile.



# Exercises

## :pencil2: Optimize the Dockerfile for Layer Caching

The `COPY . .` instruction in the original Dockerfile copies all source files before installing dependencies.
This means any change to `app.py` (or any other file) invalidates the pip install layers and forces a full reinstall.

Your task is to fix this.

- Make a trivial change to `app.py` (e.g., `echo "# my change" >> app.py`).
- Rebuild and observe that the `pip install` steps are re-executed even though the requirements didn't change.

- **Optimize the Dockerfile** by reordering the instructions so that:
   - The requirement files (`torch-requirements.txt`, `requirements.txt`) are copied first.
   - Dependencies are installed.
   - The rest of the application code is copied last.

- **Verify the fix** by rebuilding with your updated Dockerfile, making another change to `app.py` and rebuilding again.


> **The rule of thumb**: order Dockerfile instructions from **least frequently changed to most frequently changed**.

## :pencil2: Build and Push the YoloService Image

In this exercise you will build the YoloService Docker image from source, tag it, and publish it to your personal DockerHub account.


1. **Create a DockerHub account** if you don't have one:
   - Go to [hub.docker.com](https://hub.docker.com) and click **Sign up**.
   - You can sign up with your existing **GitHub account** using the "Continue with GitHub" option - no separate password needed.


2. **Build the image** using your DockerHub username as part of the image name:
   ```bash
   docker build -t <your-dockerhub-username>/yolo-service:0.0.1 .
   ```
   The full image name format for DockerHub is `<username>/<repository>:<tag>`. Using your username here means the image is already correctly named for pushing.

4. **Log in to DockerHub** using a personal access token:
   - Go to [hub.docker.com](https://hub.docker.com) → **Account Settings** → **Personal access tokens** → **Generate new token**.
   - Give the token a name (e.g. `ec2`) and set the permissions to **Read & Write**.
   - Copy the generated token — you won't be able to see it again.

   Back on your EC2 instance, log in by passing the token directly so you're never prompted interactively:
   ```bash
   docker login --username <your-dockerhub-username> --password <your-token>
   ```

5. **Push the image** to DockerHub:
   ```bash
   docker push <your-dockerhub-username>/yolo-service:0.0.1
   ```
   Docker uploads each layer separately. Layers that already exist on DockerHub (e.g., the base `python:3.11` layers) are skipped automatically.
