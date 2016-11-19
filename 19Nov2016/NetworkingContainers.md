#Docker Cairo 2016 Nov. 19 (cont.)


# Networking Containers
-------------------------

## Local networks

Docker provides a default network called `bridge` which is a special network

       $ docker network ls
       # run sample docker container which is attached to default network
       $ sudo docker run -itd --name=networktest ubuntu

You can remove a container from a network by using `disconnect` keyword using network name and container name
        
       $ sudo docker network disconnect bridge networktest        

1. Create your network

Create your own network and start attaching containers to provide complete isolation for your applications

      $ sudo docker network create -d bridge my-network
      # -d --> flag tells docker to use the bridge driver 
      $ sudo docker network ls
      # show some details about your network
      $ sudo docker network inspect my-network

2. Attach containers to your network

Attaching your app container to your network provides more secure communication to your applications.... so for example if
you want to make db communication on specific network to overcome any communication on default bridge network... you can use 
`--network` option to connect the running app to specific network as follows:
      
      # run DB container and attach it to my-network
      $ sudo docker run -d --network=my-network --name db training/postgres 
      # run web app and attach it to default bridge
      $ sudo docker run -d --name web training/webapp python app.py
      
Now your DB container is running and connected to `my-network` network... if you create your web application ... and connect it to the default bridge ... your app will going to be unreachable to your DB....This is because the two containers are running on different networks

So, you can connect the web app to multiple networks using `connect`

      $ sudo docker network connect my-bridge-network web


## Overlay networks (Global-scope network driver)

It is a logical container network which runs on top of another network, so you have to use `--driver overlay`. This is important 
because if you didn't add `--driver overlay`, it will create a local network
  
       $ sudo docker network create my-overlay-net --driver overlay
       $ sudo docker network ls 
       NETWORK ID          NAME                DRIVER              SCOPE
       a5d7b403a0a7        bridge              bridge              local               
       c4e33fba2618        docker_gwbridge     bridge              local               
       6065d82e74a6        host                host                local               
       258se3k01rbf        ingress             overlay             swarm               
       4cfakg0611de        my-overlay-net      overlay             swarm               
       4a4057542766        none                null                local

For example, when we create a docker-compose on single node, it creates a local network which is appropriate for single instance that will going to be a bridge. If we are using compose together with old swarm.. the default network is going to be overlay, so that it works across cluster.

**Important Notes:**

1. You can notice that the scope of overlay networks is swarm, which means that it is accessible across cluster nodes. Also, network IDs in bridge and host uses hex decimal identifiers, however, swarm IDs use numbers and letters.
2. Swarm uses VXLAN (Virtual Extensible LAN): a normal ethernet network, in the case of VLAN it's ethernet on top of ethernet, but in the case of VXLAN, it is ethernet on top of UDP .... which means as long as you have IP connectivity between your nodes then you will be able to have the XLAN ... so it will work on any kind of machine or any kind of cloud provider even between different cloud providers.
3. Placing container on a network must requires allocating an IP address for this container, that is because of workers cannot update Raft key/store data ... it can be updated only by managers ... so `docker run --net ... ` will run only on manager nodes.


**sources:**

1. [Docker Documentation](https://docs.docker.com/engine/tutorials/networkingcontainers/)
2. Some videos by Jérôme Petazzoni
3. [Multi-host container networking using external Key-value store](https://docs.docker.com/engine/userguide/networking/get-started-overlay/#/step-1-set-up-a-key-value-store)
4. [Docker Swarm Overlay Networks](https://docs.docker.com/engine/userguide/networking/overlay-security-model/)
