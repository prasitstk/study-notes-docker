# 02 - Using Docker Containers

## Running Docker containers

Docker CLI will send instructions to the Docker daemon.

Every commands starts with `docker`.

*Images* define what will be installed and available when the *container* is started.

The *container* is then a running instance of the *image*, which we can interact with using a shell.

To start a container from an image:

```shell
docker run <image-name>

# Example
docker run hello-world
```

Add `-it` to `docker run` will give us an insteractive shell in the started container:

```shell
docker run -it <image-name>

# Example
docker run -it ubuntu
# Then exit the shell and return to the host OS by typing:
# exit
```

Add `-d` (detached) to `docker run` will run the container in the background without printing their output to our shell.

```shell
docker run -d <image-name>

# Example
docker run -d postgres
```

List running containers by:

```shell
docker ps
```

To stop the container:

```shell
docker stop <container-ID>
```

---

## Working with Docker containers

Naming the running container by:

```shell
docker run --name <container-name> <image-name>
```

To stop the named container:

```shell
# Easier, right?
docker stop <container-name>
```

To filter running container list:

```shell
docker ps -f "name=<container-name>"

# Example
docker ps -f "name=db_pipeline_v1"
```

To view container logs:

```shell
docker logs <container-id>

# Or to follow the logs in real-time
docker logs -f <container-id>
# Ctrl+C to exit
```

The stopped container is not fully gone. It still exists and is occupying some space on our hard drive. To fully remove the stopped container because you would like to reuse its name:

```shell
docker container rm <container-id>
```

---

## Managing local docker images

**Docker Hub** is a register of community-made Docker images.

To pull an image from Docker Hub:

```shell
docker pull <image-name>
docker pull <image-name>:<image-version-called-tag>

# Example
docker pull postgres
docker pull ubuntu
docker pull ubuntu:22.04
docker pull ubuntu:jammy
```

To list the images we have available on our machine:

```shell
docker images
```

To remove an image:

```shell
docker image rm <image-name>
```

> NOTE: You can only delete an image once there are no more containers based on it.

To remove all stopped containers:

```shell
docker container prune
```

TO remove all unused images:

```shell
docker image prune -a

# Or 
docker image prune -a -f
# The -f flag is needed to avoid the confirmation prompt, feel free to leave it out when running the command yourself.
```

> NOTE: `-a` means "all". It is to remove images with unused containers and dangling images.
> NOTE: Dangling image is an image that no longer has a name because the name has been re-used for another image. This frequently occurs when creating our own updated images with the same name as the previous one an that previous one will be a dangling image.

---

## Distributing Docker Images

To pull an image from a private registry:

```shell
docker pull <registry-url>/<image-name>:<image-tag>

# Example
docker pull dockerhub.myprivateregister.com/classify_spam:v1
```

> NOTE: <private-registry-url>/<image-name>:<image-tag> is a full version of an image name.

To push an image into a registry:

```shell
docker tag <src-image-name>:<src-image-tag> <tgt-registry-url>/<tgt-image-name>:<tgt-image-tag>
docker image push <tgt-registry-url>/<tgt-image-name>:<tgt-image-tag>
```

Docker offical images -> No authentication needed

Private Docker repository -> Owner can choose

```shell
docker login <registry-url>

# Example
docker login dockerhub.myprivateregister.com
# Username:
# Password:
```

To send a Docker image to one or a few people, send it as a file:

```shell
docker save -o filename.tar <image-name>:<image-tag>

# Example: Save an image to image.tar file
docker save -o image.tar classify_spam:v1
```

To load the saved image file:

```shell
docker load -i filename.tar

# Example:
docker load -i image.tar
```

---
