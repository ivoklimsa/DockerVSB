# Docker - Linux (Part 2): Understanding the Docker File System and Volumes

We had an introduction to volumes by way of bind mounts earlier, but let's take a deeper look at the Docker file system and volumes.

The [Docker documentation](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/_) gives a great explanation on how storage works with Docker images and containers, but here's the high points.

* Images are comprised of layers
* These layers are added by each line in a Dockerfile
* Images on the same host or registry will share layers if possible
* When container is started it gets a unique writeable layer of its own to capture changes that occur while it's running
* Layers exist on the host file system in some form (usually a directory, but not always) and are managed by a [storage driver](https://docs.docker.com/engine/userguide/storagedriver/selectadriver/) to present a logical filesystem in the running container.
* When a container is removed the unique writeable layer (and everything in it) is removed as well
* To persist data (and improve performance) Volumes are used.
* Volumes (and the directories they are built on) are not managed by the storage driver, and will live on if a container is removed.  

The following exercises will help to illustrate those concepts in practice.

Let's start by looking at layers and how files written to a container are managed by something called *copy on write*.

> * [Task 1: Layers and Copy on Write](#Task1)
> * [Task 2: Understanding Docker Volumes](#Task2)
> * [Task 3: Anonymous Volumes](#Task3)
> * [Task 4: Understanding Docker Volumes](#Task4)


## <a name="Task1"></a>Task 1: Layers and Copy on Write

> Note: If you have just completed part 1 of the workshop, please close that session and start a new one.

1. In PWD click "+Add new instance" and move into that command windows.

1. Pull down the Alpine image

    ```
    $ docker pull alpine:3.11
    3.11: Pulling from library/alpine
    e6b0cf9c0882: Pull complete
    Digest: sha256:2171658620155679240babee0a7714f6509fae66898db422ad803b951257db78
    Status: Downloaded newer image for alpine:3.11
    ```

2. Pull down a Alpine example image with added command

    ```
    $ docker pull ivoklimsa/examples:alpine_example
    alpine_example: Pulling from ivoklimsa/examples
    e6b0cf9c0882: Already exists
    9a789c75f73c: Pull complete
    Digest: sha256:8118c555031d6e29eebd22194caf8be7d71d2acac44c576881ae26dc6435a210
    Status: Downloaded newer image for ivoklimsa/examples:alpine_example
    ```

    What do you notice about those the output from the Docker pull request for MySQL?

    The first layer pulled says:

    `e6b0cf9c0882: Already exists`

    Notice that the layer id (`e6b0cf9c0882`) is the same for the first layer of the Alpine_example image and the only layer in the Alpine:3.11 image. And because we already had pulled that layer when we pulled the Alpine image, we didn't have to pull it again.

    So, what does that tell us about the Alpine_example image? Since each layer is created by a line in the image's *Dockerfile*, we know that the Alpine_example image is based on the Alpine:3.11 base image. We can confirm this by looking at the [Dockerfile ](https://github.com/docker-library/mysql/blob/0590e4efd2b31ec794383f084d419dea9bc752c4/5.7/Dockerfile).

    The first line in the the Dockerfile is: `FROM alpine:3.11` This will import that layer into the Alpine_example image.

    So layers are created by Dockerfiles and are are shared between images. When you start a container, a writeable layer is added to the base image.

## <a name="Task2"></a>Task 2: Understanding Docker Volumes

[Docker volumes](https://docs.docker.com/engine/admin/volumes/volumes/) are directories on the host file system that are not managed by the storage driver. Since they are not managed by the storage drive they offer a couple of important benefits.

* **Performance**: Because the storage driver has to create the logical filesystem in the container from potentially many directories on the local host, accessing data can be slow. Especially if there is a lot of write activity to that container. In fact you should try and minimize the amount of writes that happen to the container's filesystem, and instead direct those writes to a volume

* **Persistence**: Volumes are not removed when the container is deleted. They exist until explicitly removed. This means data written to a volume can be reused by other containers.

Volumes can be anonymous or named. Anonymous volumes have no way for the to be explicitly referenced. They are almost exclusively used for performance reasons as you cannot persist data effectively with anonymous volumes. Named volumes can be explicitly referenced so they can be used to persist data and increase performance.

The next sections will cover both anonymous and named volumes.

> Special Note: These next sections were adapted from [Arun Gupta's](https://twitter.com/arungupta) excellent [tutorial](http://blog.arungupta.me/docker-mysql-persistence/) on persisting data with MySQL.

### <a name="Task3"></a>Task 3: Anonymous Volumes

1. Pull down a MySQL 5.7 image

    ```
    $ docker pull mysql:5.7
    5.7: Pulling from library/mysql
    804555ee0376: Pull complete
    c53bab458734: Pull complete
    ca9d72777f90: Pull complete
    2d7aad6cb96e: Pull complete
    8d6ca35c7908: Pull complete
    6ddae009e760: Pull complete
    327ae67bbe7b: Pull complete
    31f1f8385b27: Pull complete
    a5a3ad97e819: Pull complete
    48bede7828ac: Pull complete
    380afa2e6973: Pull complete
    Digest: sha256:b38555e593300df225daea22aeb104eed79fc80d2f064fde1e16e1804d00d0fc
    Status: Downloaded newer image for mysql:5.7
    ```

    If you look at the MySQL [Dockerfile](https://github.com/docker-library/mysql/blob/6659750146b7a6b91a96c786729b4d482cf49fe6/5.7/Dockerfile) you will find the following line:

    ```
    VOLUME /var/lib/mysql
    ```

    This line sets up an anonymous volume in order to increase database performance by avoiding sending a bunch of writes through the Docker storage driver.

    > Note: An anonymous volume is a volume that hasn't been explicitly named. This means that it's extremely difficult to use the volume later with a new container. Named volumes solve that problem, and will be covered later in this section.

2. Start a MySQL container

    ```
    $ docker run --name mysqldb -e MYSQL_USER=mysql -e MYSQL_PASSWORD=mysql -e MYSQL_DATABASE=sample -e MYSQL_ROOT_PASSWORD=supersecret -d mysql:5.7
    acf185dc16e274b2f332266a1bfc6d1df7d7b4f780e6a7ec6716b40cafa5b3c3
    ```

    When we start the container the anonymous volume is created:

3. Use Docker inspect to view the details of the anonymous volume


    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/cd79b3301df29d13a068d624467d6080354b81e34d794b615e6e93dd61f89628/_data
    ```

4. Change into the volume directory on the local host file system and list the contents #vypis!!!

    ```
    $ cd $(docker inspect -f '{{(index .Mounts 0).Source}}' mysqldb)

    $ ls
    auto.cnf            ib_buffer_pool      mysql               server-cert.pem
    ca-key.pem          ib_logfile0         performance_schema  server-key.pem
    ca.pem              ib_logfile1         private_key.pem     sys
    client-cert.pem     ibdata1             public_key.pem
    client-key.pem      ibtmp1              sample
    ```

    Notice the the directory name starts with `/var/lib/docker/volumes/` whereas for directories managed by the Overlay2 storage driver it was `/var/lib/docker/overlay2`

    As mentined anonymous volumes will not persist data between containers, they are almost always used to increase performance.

5. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

6. Create a new table

    ```
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | sample             |
    +--------------------+
    2 rows in set (0.00 sec)

    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)

    mysql> create table user(name varchar(50));
    Query OK, 0 rows affected (0.01 sec)

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

7. Exit MySQL and the MySQL container.

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

8. Stop the container and restart it

    ```
    $ docker stop mysqldb
    mysqldb

    $ docker start mysqldb
    mysqldb
    ```

9. Shell back into the running container and log into MySQL

    ```
    $ docker exec --interactive --tty mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

10. Ensure the table created previously table still exists

    ```
    mysql> connect sample;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Connection id:    4
    Current database: sample

    myslq> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

11. Exit MySQL and the MySQL container.

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

    The table persisted across container restarts, which is to be expected. In fact, it would have done this whether or not we had actually used a volume as shown in the previous section.

12. Let's look at the volume again

    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/cd79b3301df29d13a068d624467d6080354b81e34d794b615e6e93dd61f89628/_data
    ```

    We do see the volume was not affected by the container restart either.

    Where people often get confused is in expecting that the anonymous volume can be used to persist data BETWEEN containers.

    To examine that delete the old container, create a new one with the same command, and check to see if the table exists.

13. Remove the current MySQL container

    ```
    $ docker container rm --force mysqldb
    mysqldb
    ```

14. Start a new container with the same command that was used before

    ```
    $ docker run --name mysqldb -e MYSQL_USER=mysql -e MYSQL_PASSWORD=mysql -e MYSQL_DATABASE=sample -e MYSQL_ROOT_PASSWORD=supersecret -d mysql:5.7
    eb15eb4ecd26d7814a8da3bb27cee1a23304fab1961358dd904db37c061d3798
    ```

15. List out the volume details for the new container

    ```
    $ docker inspect -f 'in the {{.Name}} container {{(index .Mounts 0).Destination}} is mapped to {{(index .Mounts 0).Source}}' mysqldb
    in the /mysqldb container /var/lib/mysql is mapped to /var/lib/docker/volumes/e0ffdc6b4e0cfc6e795b83cece06b5b807e6af1b52c9d0b787e38a48e159404a/_data
    ```

    Notice this directory is different than before.

16. Shell back into the running container and log into MySQL

    ```
    $ docker exec --interactive --tty mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

17. Check to see if table created previously table still exists

    ```
    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)
    ```

18. Exit MySQL and the MySQL container.

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

19. Remove the container

    ```
    docker container rm --force mysqldb
    mysqldb
    ```

So while a volume was used to store the new table in the original container, because it wasn't a named volume the data could not be persisted between containers.

To achieve persistence a named volume should be used.

### <a name="Task4"></a>Task 4: Named Volumes

A named volume (as the name implies) is a volume that's been explicitly named and can easily be referenced.

A named volume can be create on the command line, in a docker-compose file, and when you start a new container. They [CANNOT be created as part of the image's dockerfile](https://github.com/moby/moby/issues/30647).

1. Start a MySQL container with a named volume (`dbdata`)

    ```
    $ docker run --name mysqldb \
    -e MYSQL_USER=mysql \
    -e MYSQL_PASSWORD=mysql \
    -e MYSQL_DATABASE=sample \
    -e MYSQL_ROOT_PASSWORD=supersecret \
    --detach \
    --mount type=volume,source=mydbdata,target=/var/lib/mysql \
    mysql:5.7
    ```

    Because the newly created volume is empty, Docker will copy over whatever existed in the container at `/var/lib/mysql` when the container starts.

    Docker volumes are primatives just like images and containers. As such, they can be listed and removed in the same way.

2. List the volumes on the Docker host

    ```
    $ docker volume ls
    DRIVER              VOLUME NAME
    local               55c322b9c4a644a5284ccb5e4d7b6b466a0534e26d57c9ef4221637d39cf9a88
    local               cc44059d23e0a914d4390ea860fd35b2acdaa480e83c025fb381da187b652a66
    local               e0ffdc6b4e0cfc6e795b83cece06b5b807e6af1b52c9d0b787e38a48e159404a
    local               mydbdata
    ```

3. Inspect the volume

    ```
    $ docker inspect mydbdata
    [
        {
            "CreatedAt": "2017-10-13T19:55:10Z",
            "Driver": "local",
            "Labels": null,
            "Mountpoint": "/var/lib/docker/volumes/mydbdata/_data",
            "Name": "mydbdata",
            "Options": {},
            "Scope": "local"
        }
    ]
    ```

    Any data written to `/var/lib/mysql` in the container will be rerouted to `/var/lib/docker/volumes/mydbdata/_data` instead.

4. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

5. Create a new table

    ```
    mysql> connect sample;
    Connection id:    4
    Current database: sample

    mysql> show tables;
    Empty set (0.00 sec)

    mysql> create table user(name varchar(50));
    Query OK, 0 rows affected (0.01 sec)

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

6. Exit MySQL and the MySQL container.

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

7. Remove the MySQL container

    ```
    $ docker container rm --force mysqldb
    ```

    Because the MySQL was writing out to a named volume, we can start a new container with the same data.

    When the container starts it will not overwrite existing data in a volume. So the data created in the previous steps will be left intact and mounted into the new container.

8. Start a new MySQL container

    ```
    $ docker run --name new_mysqldb \
    -e MYSQL_USER=mysql \
    -e MYSQL_PASSWORD=mysql \
    -e MYSQL_DATABASE=sample \
    -e MYSQL_ROOT_PASSWORD=supersecret \
    --detach \
    --mount type=volume,source=mydbdata,target=/var/lib/mysql \
    mysql:5.7
    ```

9. Shell into your running MySQL container and log into MySQL

    ```
    $ docker exec --tty --interactive new_mysqldb bash

    root@132f4b3ec0dc:/# mysql --user=mysql --password=mysql
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.19 MySQL Community Server (GPL)

    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

10. Check to see if the previously created table exists in your new container.

    ```
    mysql> connect sample;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Connection id:    4
    Current database: sample

    mysql> show tables;
    +------------------+
    | Tables_in_sample |
    +------------------+
    | user             |
    +------------------+
    1 row in set (0.00 sec)
    ```

    The data will exist until the volume is explicitly deleted.

11. Exit MySQL and the MySQL container.

    ```
    mysql> exit
    Bye

    root@132f4b3ec0dc:/# exit
    exit
    ```

12. Remove the new MySQL container and volume

    ```
    $ docker container rm --force new_mysqldb
    new_mysqldb

    $ docker volume rm mydbdata
    mydbdata
    ```

    If a new container was started with the previous command, it would create a new empty volume.
