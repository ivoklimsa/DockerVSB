# Docker - Linux (Part 3): Docker Container Networking

So far all of the previous exercises have been based around running a single container on a single host.

This section will provide an overview of Docker's multi-host networking capabilities.

#### Networking

Docker supports several different networking options, but this lab will cover the two most popular: bridge and overlay.

Bridge networks are only available on the local host, and can be created on hosts in swarm clusters as well as standalone hosts. However, in a swarm cluster, even though the machines are tied together, bridge networks only work on the host on which they were created.

Overlay networks facilitate the creation of networks that span Docker hosts. While it's possible to network together hosts that are not in a Swarm cluster, it's a very manual task requiring the addition of an external key value store. With Docker swarm creating overlay networks is trivial.

> * [Task 1: Bridge networking](#Task_1)
> * [Task 2: Overlay Networking](#Task_2)

## <a name="task1"></a>Task 1: Bridge networking


### Bridge networking overview

As previously mentioned, bridge networks facilitate the create of software-defined networks on a single Docker host. Containers on one bridge network cannot communicate with containers on a different bridge network (unless they communicate via the network the Docker host resides on).

1. On `node1` create a bridge network (`mybridge`)

    ```
    $ docker network create mybridge
    52fb9de4ad1cbe505e451599df2cb62c53e56893b0c2b8d9b8715b5e76947551
    ```

2. List the networks on `node1`

    ```
    $ docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    edf9dc771fc4        bridge              bridge              local
    e5702f60b7c9        docker_gwbridge     bridge              local
    7d6b733ee498        host                host                local
    rnyatjul3qhn        ingress             overlay             swarm
    52fb9de4ad1c        mybridge            bridge              local
    dbd52ffda3ae        none                null                local
    ```

    The newly created `mybridge` network is listed.

    > Note: Docker creates several networks by default, however the purpose of those networks is outside the scope of this workshop.

3. Switch to `node2`

4. List the available networks on `node2`

    ```
    $ docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    3bc2a78be20f        bridge              bridge              local
    641bdc72dc8b        docker_gwbridge     bridge              local
    a5ef170a2758        host                host                local
    rnyatjul3qhn        ingress             overlay             swarm
    3dec80db87e4        none                null                local
    ```

    Notice that the same networks names exist on `node2` but their ID's are different. And, `mybridge` does not show up at all.

5. Move back to `node1`

6. Create an Alpine container named `alpine_host` running the `top` process in `detached` mode and connecit it to the `mybridge` network.

    ```
    $ docker container run \
      --detach \
      --network mybridge \
      --name alpine_host \
      alpine top
    Unable to find image 'alpine:latest' locally
    latest: Pulling from library/alpine88286f41530e: Pull complete
    Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
    Status: Downloaded newer image for alpine:latest
    974903580c3e452237835403bf3a210afad2ad1dff3e0b90f6d421733c2e05e6
    ```
    > Note: We run the `top` process to keep the container from exiting as soon as it's created.

7. Start another Alpine container named `alpine client`

    ```
    $ docker container run \
      --detach \
      --name alpine_client \
      alpine top
    c81a3a14f43fed93b6ce2eb10338c1749fde0fe7466a672f6d45e11fb3515536
    ```

8. Attempt to PING `alpine_host` from `alpine_client`

    ```
    $ docker exec alpine_client ping alpine_host
    ping: bad address 'alpine_host'
    ```

    Because the two containers are not on the same network they cannot reach each other.

9. Inspect `alpine_host` and `alpine_client` to see which networks they are attached to.

    ```
    $ docker inspect -f {{.NetworkSettings.Networks}} alpine_host
    map[mybridge:0xc420466000]

    $ docker inspect -f {{.NetworkSettings.Networks}} alpine_client
    map[bridge:0xc4204420c0]
    ```

    `alpine_host` is, as expected, attached to the `mybridge` network.

    `alpine_client` is attached to the default bridge network `bridge`

10. Stop and remove `alpine_client`

    ```
    $ docker container rm --force alpine_client
    alpine_client
    ```

11. Start another container called `alpine_client` but attach it to the `mybridge` network this time.

    ```
    $ docker container run \
      --detach \
      --network mybridge \
      --name alpine_client \
      alpine top
    8cf39f89560fa8b0f6438222b4c5e3fe53bdeab8133cb59038650231f3744a79
    ```

12. Verify via `inspect` that `alpine_client` is on the `mybridge` network

    ```
    $ docker inspect -f {{.NetworkSettings.Networks}} alpine_client
    map[mybridge:0xc42043e0c0]
    ```

13. PING `alpine_host` from `alpine_client`

    ```
    docker exec alpine_client ping -c 5 alpine_host
    PING alpine_host (172.20.0.2): 56 data bytes
    64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.102 ms
    64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.108 ms
    64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.088 ms
    64 bytes from 172.20.0.2: seq=3 ttl=64 time=0.113 ms
    64 bytes from 172.20.0.2: seq=4 ttl=64 time=0.122 ms

    --- alpine_host ping statistics ---
    5 packets transmitted, 5 packets received, 0% packet loss
    round-trip min/avg/max = 0.088/0.106/0.122 ms
    ```

    Something to notice is that it was not necessary to specify an IP address.  Docker has a built in DNS that resolved `alpine_client` to the correct address.

Being able to network containers on a single host is not extremely useful. It might be fine for a simple test envrionment, but production environments require the ability provide the scalability and fault tolerance that comes from having multiple interconnected hosts.

This is where overlay networking comes in.

## <a name="task2"></a>Task 2: Overlay Networking

### Overlay Networking Overview

Overlay networks in Docker are software defined networks that span multiple hosts (unlike a bridge network which is limited to a single Docker host). This allows containers on different hosts to easily communicate on the Docker networking fabric (vs having to move out to the host's network).

This next section covers building an overlay network and having two containers communicate with each other.

1. Remove the existing Alpine containers

    ```
    $ docker container rm --force $(docker ps --all --quiet)
    e65629beeb57
    5cc5eeaf703b
    ```

2. Create a new overlay network (`-d` specifies the networking driver to use, if it's omitted `bridge` is the default).

    ```
    $ docker network create --attachable -d overlay myoverlay
    z16nhzxwbeukjnz3e6nk2159p
    ```

    > Note: We have to use the `--attachable` flag because by default you cannot use `docker run` on overlay networks that are part of a swarm. The preferred method is to use a Docker *service* which is covered later in the workshop.

3. List the networks on the host to verify that the `myoverlay` network was created.

    ```
    $ docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    edf9dc771fc4        bridge              bridge              local
    e5702f60b7c9        docker_gwbridge     bridge              local
    7d6b733ee498        host                host                local
    rnyatjul3qhn        ingress             overlay             swarm
    52fb9de4ad1c        mybridge            bridge              local
    z16nhzxwbeuk        myoverlay           overlay             swarm
    dbd52ffda3ae        none                null                local
    ```

3. Create an Alpine container and attach it to the `myoverlay` network.

    ```
    $ docker container run \
      --detach \
      --network myoverlay \
      --name alpine_host \
      alpine top
    a604aa48660835aeec75f3239964d35c334bcdf33d1b5574c319aaf344c2119a
    ```

4. Move to `node2`

5. List the available networks

    ```
    $ docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    3bc2a78be20f        bridge              bridge              local
    641bdc72dc8b        docker_gwbridge     bridge              local
    a5ef170a2758        host                host                local
    rnyatjul3qhn        ingress             overlay             swarm
    3dec80db87e4        none                null                local
    ```

    Notice anything out of the ordinary? Where's the `myoverlay` network?

    Docker won't extend the network to hosts where it's not needed. In this case, there are no containers attached to `myoverlay` on `node2` so the network has not been extended to the host.

6. Start an alpine container and attach it to `myoverlay`

    ```
    $ docker container run \
      --detach \
      --network myoverlay \
      --name alpine_client \
      alpine top
    Unable to find image 'alpine:latest' locally
    latest: Pulling from library/alpine
    88286f41530e: Pull complete
    Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
    Status: Downloaded newer image for alpine:latest
    5d67e360d8e42c618dc8ea40ecd745280a8002652c7bcdc7982cb5c6cdd4fd13
    ```

7. List the available networks on `node2`

    ```
    $ docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    3bc2a78be20f        bridge              bridge              local
    641bdc72dc8b        docker_gwbridge     bridge              local
    a5ef170a2758        host                host                local
    rnyatjul3qhn        ingress             overlay             swarm
    z2fh5l7g1b4k        myoverlay           overlay             swarm
    3dec80db87e4        none                null                local
    ```

    The `myoverlay` network is now available on `node2`

8. Ping `apine_host`

    ```
    $ docker exec alpine_client ping -c 5 alpine_host
    PING alpine_host (10.0.0.2): 56 data bytes
    64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.244 ms
    64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.122 ms
    64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.166 ms
    64 bytes from 10.0.0.2: seq=3 ttl=64 time=0.201 ms
    64 bytes from 10.0.0.2: seq=4 ttl=64 time=0.137 ms
    ```
    Networking also works betwen Linux and Windows nodes

9. Move to the `node3`

10. Ping `alpine_host` from the `node3`

    ```
    docker container run \
      --rm \
      --network myoverlay \
      alpine ping alpine_host

    Pinging alpine_host [10.0.0.2] with 32 bytes of data:
    Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
    Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
    Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
    Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
    ```

    > Note: In some cases it may take a few seconds for the client to find the alpine host resutling in PING timeouts. If this happens, simply retry the above command.
