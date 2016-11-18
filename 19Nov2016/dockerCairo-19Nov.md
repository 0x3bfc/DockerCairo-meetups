# Docker Cairo 2016 Nov. 19


Prerequests:
------------

1. Install three nodes on Windows Azure cloud
	
	- master
	- node1
	- node2
	

source [docker documentation](https://docs.docker.com/swarm/install-manual/)

Install Engine on Each node
----------------------------

      $ sudo apt-get update
      $ curl -sSL https://get.docker.com/ | sh

Edit /etc/sysconfig/docker and add "-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"  to the OPTIONS variable.
      
      $ sudo vi /etc/default/docker
      
        ..........
        DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"
        ..........
        ..........
        
      $ sudo /etc/init.d/docker restart
      $ sudo usermod -aG docker <USERNAME>
      
      $ sudo docker pull swarm
      $ sudo docker pull progrium/consul
      
     
     
Discovery service
-----------------

       consul discovery backend

       $ sudo docker run -d -p 8500:8500 --name=consul progrium/consul -server -bootstrap


Start Swarm Cluster
-------------------

- Start swarm manager0 & manager1 (secondary manager for high availability) 

       $ sudo docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <Manager_0>:4000 consul://<manager_0>:8500
       $ sudo docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <Manager_1>:4000 consul://<manager_0>:8500
       
- Connect to node1 and node2 to swarm and join them to the cluster

       $ sudo docker run -d swarm join --advertise=<node_0>:2375 consul://<manager_0>:8500
       


Get manager and node status
---------------------------

       $ sudo docker -H :4000 info
       
       

ISSUES:
-------

- Don't duplicate the VM or docker daemon installation

The problem is that docker generates a single ID when you install the daemon. 

If you duplicate the VM you will end up with two different hosts and their own IP address but with a duplicate ID from docker that swarm uses. 
You should reinstall the docker daemon on the second node

source: https://github.com/docker/swarm/issues/563



Node management https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/
Swarm networking https://docs.docker.com/swarm/networking/
