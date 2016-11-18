# Docker Cairo 2016 Nov. 19

Deploy your swarm cluster as described below. The cluster was deployed on Azure cloud. If you have any issue, don't 
hesitate to ask!!

Prerequisite:
-------------

1. Install three nodes on Windows Azure cloud

   - manager0 
   - manager1 (secondary)
   - node1
   - node2
	

  source [docker documentation](https://docs.docker.com/swarm/install-manual/)

2. Install Engine on Each node

         $ sudo apt-get update
         $ curl -sSL https://get.docker.com/ | sh

3. Configure Docker engine 

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
     
4. Discovery service (Consul)

         $ sudo docker run -d -p 8500:8500 --name=consul progrium/consul -server -bootstrap


5. Start Swarm Cluster

 Start swarm manager0 & manager1 (secondary manager for high availability) 

       $ sudo docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <Manager_0>:4000 consul://<manager_0>:8500
       $ sudo docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <Manager_1>:4000 consul://<manager_0>:8500

  Connect to node1 and node2 to swarm and join them to the cluster

       $ sudo docker run -d swarm join --advertise=<node_0>:2375 consul://<manager_0>:8500

6. Check manager and node status

        $ sudo docker -H :4000 info
       
       

NOTES
------

Don't duplicate the VM or docker daemon installation

The problem is that docker generates a single ID when you install the daemon. 

If you duplicate the VM you will end up with two different hosts and their own IP address but with a duplicate ID from docker that swarm uses. 
You should reinstall the docker daemon on the second node

source: [github issue](https://github.com/docker/swarm/issues/563)



# Node Management
-------------------
In this tutorial we are going to talk about Docker swarm node management. As we can see basic architecture as described below, we have three managers and unlimited number of workers. All workers are connected to a gossip network!

![Alt text](images/swarm-diagram.png "Basic Swarm cluster Architecture")

Swarm enables docker users to manage multiple docker-engines on multiple physical/virtual hosts.... This leads to using 
different techniques such as leader election, failure detection, suspicion and consensus mechanisms to manage large number 
of services running on swarm cluster.

1. Managers
------------

Managers are used to **maintaining cluster state** by implementing [RAFT](https://raft.github.io/raft.pdf) consensus algorithm. To satisty the high availability of service, we have to replicate our services but we need a leader to coordinate communication among distributed servers. 

### Leader Election:

- Leader election algorithm must satisfy the following:
   
   1. Elect one leader only among the **non-faulty processes**
   2. All non-faulty processes agree on who is the leader
   
- Most popular Leader election algorithms:
 
The most popular election algorithms can be classified into two main categories classical leader election protocols such
as (Ring protocols/algorithms) and Paxos-like approaches such as Google CHUBBY and zookeeper. 

   1. **Ring Election** 
   
      ![Alt text](images/Ring-LeaderElection.png "Ring Leader Election Algorithm")
      
      * N processes/nodes are organized in a logical ring.
      * Any node can initiate election in case of leader failure.
      * Each node send message to its successor containing the leader ID.
      * If the current Node has a higher ID .. change the leader ID to current node ID.
      * until the initiator recieve the highest ID then send a message to all node containing the elected leader.
      
      
   2. **Google CHUBBY**
   
   - group of replicas need to have a master by following:
   
      ![Alt text](images/google-chu.png "Google Chubby Leader Election Algorithm")
      
      * Potential leader tries to get votes from other servers
      * Each server votes for at most one leader
      * Server with majority of votes becomes new leader, informs everyone
      * Master node run election again after a period called *Master lease*
      
      
   3. **Zookeeper**
   
      * Each Server monitors its next higher server id
      * If that successor was the leader and it has failed .... become a new leader
      * Else: wait timeout and check successor again.

   4. **RAFT**
   
source: https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/


Swarm networking https://docs.docker.com/swarm/networking/
