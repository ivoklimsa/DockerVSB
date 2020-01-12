# Docker - Linux (Part 1 ): Introduction and Setup

In this lab we'll setup access for DockerID and LinuxVMs.


> * [Task 1: Prerequisites](#Task1)
> * [Task 2: Simple Docker container](#Task2)


> Note: Docker CLI Cheatsheet  https://devhints.io/docker

## <a name="Task1"></a>Task 1: Prerequisites

Before we start, you'll need to gain access to your Linux VM, clone a GitHub repo, and make sure you have a DockerID.

### Make sure you have a DockerID

If you do not have a DockerID (a free login used to access Docker Cloud, Docker Store, and Docker Hub), please visit [Docker Cloud](https://cloud.docker.com) to register for one.


### Access your Linux VM
1. Visit [Play With Docker](https://labs.play-with-docker.com)
2. Click `Start Session`
2. On the left click `+ Add New Instance`

All of the exercises in this lab will be performed in the console window on the right of the Play With Docker screen.

### Clone the Lab GitHub Repo

Use the following command to clone the lab repo from GitHub.

```
$ git clone https://github.com/ivoklimsa/DockerVSB.git
Cloning into 'DockerMeetupAdvanced'...
remote: Counting objects: 88, done.
remote: Compressing objects: 100% (40/40), done.
remote: Total 88 (delta 33), reused 88 (delta 33), pack-reused 0
Unpacking objects: 100% (88/88), done.
```

## <a name="Task2"></a>Task 2: Run some simple Docker containers

In this section you'll deploy simple Docker container.

### Run a single-task Alpine Linux container

In this step we're going to start a new container and tell it to run the `hostname` command. The container will start, execute the `hostname` command, then exit.

Run the following command in your Linux console:

    ```
    $ docker container run alpine hostname
    Unable to find image 'alpine:latest' locally
    latest: Pulling from library/alpine
    88286f41530e: Pull complete
    Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
    Status: Downloaded newer image for alpine:latest
    888e89a3b36b
    ```

    The output above shows that the `alpine:latest` image could not be found locally. When this happens, Docker automatically *pulls* it form Docker Hub.

    After the image is pulled, the container's hostname is displayed (`888e89a3b36b` in the example above).
