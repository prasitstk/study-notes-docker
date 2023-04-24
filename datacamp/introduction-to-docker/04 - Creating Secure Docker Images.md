# 04 - Creating Secure Docker Images

## Changing users and working directory

`FROM`, `RUN`, and `COPY` instructions only affect **the file system**, not each other.

However, some instructions can influence other instructions directly. 

- `WORKDIR`: changes **the working directory** for *the following instructions*.
- `USER`: changes **which user is executing** *the following instructions*.

### `WORKDIR` - Changing the working directory

When using a Dockerfile instruction where we have to specify **a path**, we can always use **a full path**.
When working with long paths, this can quickly become **hard to read**.

The alternative to using full paths is the `WORKDIR` instruction, which allows us to change the directory inside the image from which *all subsequent instructions* will be executed. 

For example, with `COPY`:

From:

```Dockerfile
COPY /projects/pipeline_v3/ /home/my_user_with_a_long_name/work/projects/app/
```

To:

```Dockerfile
WORKDIR /home/my_user_with_a_long_name/work/projects/
COPY /projects/pipeline_v3/ app/
```

For example, with `RUN`:

From:

```Dockerfile
RUN /home/repl/projects/pipeline/init.sh
RUN /home/repl/projects/pipeline/start.sh
```

To:

```Dockerfile
WORKDIR /home/repl/projects/pipeline/
RUN ./init.sh
RUN ./start.sh
```

For example, with `CMD`:

From:

```Dockerfile
CMD /home/repl/projects/pipeline/start.sh
```

To:

```Dockerfile
WORKDIR /home/repl/projects/pipeline/
CMD start.sh
```

Also overriding command will be run in `WORKDIR`:

```shell
docker run -it pipeline_image start.sh
```

### Linux permissions

- Permissions are assigned to users.
- **root** is a special user with all permissions.

### Best practice

- Use **root** to create new users with permissions for specific tasks.
- STOP using **root**.

The image we start our **Dockerfile** from will determine the user. 

`ubuntu` -> **root** by default

```Dockerfile
FROM ubuntu        # Root user by default
RUN apt-get update # Run as root
```

This has the advantage that all folders are accessible, and we won't get errors about permissions when installing anything. However, it is unsafe as all instructions will run with full permissions.

Use `USER` instruction:

```Dockerfile
FROM ubuntu        # Root user by default
USER repl          # Changes the user to repl
RUN apt-get update # Run as repl
```

The `USER` **Dockerfile** instruction allow us to change the user in the **image**. 
Any *following instructions* will be run as the user set by the `USER` instruction. 

So, it can be *used multiple times*, and the *latest instruction* will determine the user executing *the following instructions*.

The **last** `USER` instruction in a **Dockerfile** will also control the user in any **containers** started from the image of this Dockerfile.

```shell
docker run -it ubuntu bash
# repl@container: whoami
# repl
```

---

## Variables in Dockerfiles

Using variables in our Dockerfiles allows us to make them less verbose, safer to change, and easier to update.

### Variables with the `ARG` instruction

The `ARG` instruction allows us to set variables **inside a Dockerfile** and then use that variable **throughout the Dockerfile**.

```Dockerfile
ARG <var-name>=<var-value>

# For example
ARG path=/home/repl

# To use in the Dockerfile
COPY /local/path $path
```

However, the variable won't be accessible after the image is built. 
In other words, the variable defined using `ARG` will **NOT exist inside the container**.

### Use-cases for the `ARG` instruction

Define a version we need in multiple places throughout the Dockerfile.

```Dockerfile
FROM ubuntu
ARG python_version=3.9.7-1+bionic1
RUN apt-get install python3=$python_version
RUN apt-get install python3-dev=$python_version
```

Define a path to a project or user directory.

```Dockerfile
FROM ubuntu
ARG project_folder=/projects/pipeline_v3
COPY /local/project/files $project_folder
COPY /local/project/test_files $project_folder/tests
```

### Setting ARG variables at build time

From:

```Dockerfile
FROM ubuntu
ARG project_folder /projects/pipeline_v3
COPY /local/project/files $project_folder
COPY /local/project/test_files $project_folder/tests
```

Set a variable in the `build` command:

```shell
docker build --build-arg project_folder=/repl/pipeline .
```

`ARG` is overwritten and files end up in:

```Dockerfile
FROM ubuntu
COPY /local/project/files /repl/pipeline
COPY /local/project/test_files /repl/pipeline/tests
```

### Variables with ENV

The second way to define variables in Dockerfiles is by using the `ENV` instruction. 

```Dockerfile
ENV <var-name>=<var-value>

# For example
ENV DB_USER=pipeline_user

# To use in the Dockerfile or at runtime
CMD psql -U $DB_USER
```

Unlike the `ARG` instruction, variables set with `ENV` are **still accessible after the image is built**.

- ARG are used to change the behavior of **Dockerfiles** *during the build*
- ENV are used to change behavior of both **Dockerfiles** *during the build* and **containers** *at runtime*.

### Use-cases for the ENV instruction

Setting a directory to be used at runtime:

```Dockerfile
ENV DATA_DIR=/usr/local/var/postgres

ENV MODE production
```

Unlike `ARG` variables, it is **not possible to override** `ENV` variables at *build time*. 

However, it is **possible to override** `ENV` variables *when starting a container* from an image.
This can be done using the `--env` parameter of the `docker run` command:

Setting or replacing a variable at *runtime*:

```shell
docker run --env <key>=<value> <image-name>

# For example
docker run --env POSTGRES_USER=test_db --env POSTGRES_PASSWORD=test_db postgres
```

### Secrets in variables are not secure

Both `ENV` and `ARG` variables seem convenient for adding passwords or other secrets to a docker image at build or runtime. 
However, both are **NOT secure** to use for secrets.

Anyone can look at *variables defined in a Dockerfile* **after the image is built** with the `docker history` command.

```shell
docker history <image-name>
```

- This command shows all the steps that were done to build an image.

If, instead, we *pass variables at build or start time*, they can be also found in the bash `history` of the host or image.

Keep in mind that if we use secrets to create our image without using **more advanced techniques to hide them**, they will be shared with anybody we share the image with.

---

## Creating Secure Docker Images

### Inherent Security

Docker **inherently provides more security over running applications locally** because there is **an extra layer of isolation** between the *application* and *our host operating system*. 

This makes it much safer to open an application or archive from an unknown source in a container in comparison to doing the same on your local machine. 

However, that doesn't mean it is 100% safe to do so. A malicious payload **can escape the container's isolation** and **infect the host**.

> NOTE: Attackers breaking out of *a container to the host operating system* is the main risk of using containers.
> NOTE: However, using a container is STILL a good way to run an executable or open an archive from an untrusted/unknown source because you greatly decrease the chance of a malicious actor accessing your computer. In other words, it is not 100% safe because a malicious actor can break out of a container, but luckily this chance is very small.

### Making secure images :: Images from a trusted source

Choose the right image to start from.

Docker Hub filters (Trusted Content):

- Docker Official Image
- Verified Publisher
- Sponsored OSS

All three Trusted Content filters will give us images we consider safe for the most use-cases.

### Making secure images :: Keep software up-to-date

Even images downloaded from the official Docker Hub Repository **aren't always up-to-date**.

While it could be the case no safety-related updates have been made to anything installed in these images since then, 
best practice is to **update the software to its latest version in images** ourselves.

For example:

```Dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get upgrade -y
```

### Making secure images :: Keep images minimal

Q: What's better than ensuring all software in our image is updated? 
A: Having less of it. 

- This also means we will have to keep less software up to date.

TO sum, when creating a secure image, ensure you **ONLY install the software you need** for its current use case. 

### Making secure images :: Don't run applications as root

**Allowing `root` access to an image** defeats *keeping the image up-to-date* and *minimal* because this will allow anybody who gets access to a container to **install anything they want**.

Instead, make **containers start as a user with fewer permissions**. For example:

```Dockerfile
FROM ubuntu                  # User is set to root by default.
RUN apt-get update
RUN apt-get install python3
USER repl                    # We switch the user after installing what we need for our use-case.
CMD python3 pipeline.py
```

This way we can ensure that any malicious code in the pipeline does not have `root` access in our container.

---
