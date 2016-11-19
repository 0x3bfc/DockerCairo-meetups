# Docker Cairo 2016 Nov. 19 (cont.)

Deploy your swarm cluster as described below. The cluster was deployed on Azure cloud. If you have any issues, don't
hesitate to ask!!

# Prerequisites:
----------------

1. Install four nodes on Windows Azure cloud

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

  Connect node1 and node2 to swarm and join them to the cluster

       $ sudo docker run -d swarm join --advertise=<node_0>:2375 consul://<manager_0>:8500

6. Check manager and node status

        $ sudo docker -H :4000 info


# Node Management
-------------------

In this tutorial we are going to talk about Docker swarm node management. As we can see basic architecture as described below, we have three managers and unlimited number of workers. All workers are connected to the same network!

![Alt text](images/swarm-diagram.png "Basic Swarm cluster Architecture")
source: https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/

Swarm enables docker users to manage multiple docker-engines on multiple physical/virtual hosts.... This leads to using
different techniques such as leader election, failure detection, suspicion and consensus mechanisms to manage large number
of services running on swarm cluster.


1. Managers
------------

* Managers are used to **maintaining cluster state** by implementing [RAFT](https://raft.github.io/raft.pdf) consensus algorithm.
* To satisty the high availability of service, we have to replicate our services but we need a leader to coordinate communication among distributed servers.
* Docker recommends a maximum of **seven manager nodes** for a swarm!, but this doesn't increase the performance.

2. Workers
------------

* Worker nodes mainly execute containers using Docker Engine.
* You have to deploy at least one manager to be able to start worker
* By default, all managers are defined as workers.
* As a result, you can set the availabily value to "drain" in order to prevent scheduler from setting tasks on managers


# Operations on nodes
----------------------

   **Node Availability**
   ---------------------

   There are three types of node availability:

   * **Active** = schedule tasks on this node
   * **Pause**  = doesn't schedule tasks on this node, but existing tasks are not effected!
   * **Drain**  = doesn't schedule tasks on this node, existing tasks are moved away

   For example, change availability status of node-1 from active mode to drain:

            $ sudo docker node update --availability drain node-1

   **List Nodes**
   ---------------------
	Run the following commands on manager:

		   $ sudo docker node ls

    **Inspect node**
    ----------------

		   $ sudo docker node inspect baqteepnex4d2wuxnd0bresu0 --pretty
		   ID:                     baqteepnex4d2wuxnd0bresu0
		   Hostname:               dockercairo
		   Joined at:              2016-11-18 14:56:56.91974244 +0000 utc
		   Status:
			State:                 Ready
		   Availability:          Active
		   Manager Status:
		   Address:               100.109.108.132:2377
		   Raft Status:           Reachable
			Leader:                Yes
		   Platform:
			Operating System:      linux
			Architecture:          x86_64
		   Resources:
			CPUs:                  4
			Memory:                6.804 GiB
		   Plugins:
			Network:              bridge, host, null, overlay
			Volume:               local
		   Engine Version:         1.12.3

		   # show self inspect
		   $ sudo docker node inspect self --pretty

   **Update node**
   ---------------

	update node label

		   $ sudo docker node update --label-add foo --label-add bar=baz aun655dx7djrr4x06il5c7g46

	To prevent the scheduler from placing tasks on a manager node in a multi-node swarm, set the availability for the manager node to Drain.

		   $ sudo docker node update --availability drain baqteepnex4d2wuxnd0bresu0

		   $ sudo docker node ls
			ID                           HOSTNAME               STATUS  AVAILABILITY  MANAGER STATUS
			aun655dx7djrr4x06il5c7g46    localhost.localdomain  Ready   Active
			baqteepnex4d2wuxnd0bresu0 *  dockercairo            Ready   Drain         Leader
			bfhbnpelvbu9igdm1yr77ep58    worker2                Ready   Active

   **Change node role**
   --------------------

	In case of manager maintaince, You can make worker node acts as manager using "promote" and "demote".

	Nodes can be promoted to manager :

			$ sudo docker node promote <NODE ID>


	Also, nodes can be demoted to worker with docker demote

			$ sudo docker node demote <NODE ID>

	This can also be done with `docker node update <NODE ID> --role <worker|manager>`
	but this has to be done from manager node ... workers cannot promote themselves.

   **Removing nodes**
   --------------------
   swarm enables nodes to leave swarm cluster using simple swarm leave option Run the following command on node **NOT MANAGER**

	   $ sudo docker swarm leave

   * Manager can not leave
   * Nodes are drained before being removed (Rescheduling all tasks on other workers).
   * After leaving, a node shows up in `docker node ls`
   * If you run `docker node ls` again, you will notice that the node has no status, but still exist ... that because
   when node leave swarm, the id will be exist, so to remove it completely, you have to use `docker node rm <NODE ID>`
   fromm manager.

   **Deploy & scale a serveice**
   ----------------------------

           $ sudo docker service create --replicas 2 --name helloworld alpine ping docker.com
	   $ sudo docker -H :4000 ps
	   # scale service
	   $ sudo docker service scale helloworld=4


Docker Swarm load balancing
------
Swarm uses scheduling capabilities to ensure there are sufficient resources for
distributed containers. Swarm assigns containers to underlying nodes and
optimizes resources by automatically scheduling container workloads to run on
the most appropriate host. This Docker orchestration balances containerized
application workloads, ensuring containers are launched on systems with adequate
resources, while maintaining necessary performance levels.

    Swarm uses three different strategies to determine on which nodes each
    container should run:

    * Spread -- Acts as the default setting and balances containers across the
    nodes in a cluster based on the nodes' available CPU and RAM, as well as
    the number of containers it is currently running. The benefit of the Spread
    strategy is, if the node fails, only a few containers are lost.
    * BinPack -- Schedules containers to fully use each node. Once a node is full, it moves on to the next in the cluster. The benefit of BinPack is it uses a smaller amount of infrastructure and leaves more space for larger containers on unused machines.
    * Random -- Chooses a node at random.
NOTES
------

* Don't duplicate the VM or docker daemon installation. The problem is that docker generates a single ID when you install the daemon. If you duplicate the VM you will end up with two different hosts and their own IP address but with a duplicate ID from docker that swarm uses. You should reinstall the docker daemon on the second node ... source: [github issue](https://github.com/docker/swarm/issues/563)
* Regarding Consul: to create a high-availability cluster use a trio of consul nodes... for more info check out this [link](https://hub.docker.com/r/progrium/consul/)
* Also *--advertise-addr* enable node to propagate that information to other nodes that subsequently connect to it.


**Sources**

1. [Docker Documentation](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/)
2. [Trio Consul Nodes](https://hub.docker.com/r/progrium/consul/)
