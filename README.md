# cl_netq_demo_k8s
Cumulus Vagrant file and scripts for NetQ with Kubernetes Demo

This Github repository contains the Vagrantfile and scripts necessary to set up a typical data center topology using Cumulus Linux.  It clones a playbook to set up a Layer 3 BGP environment for container networking and uses Kubernetes (k8s) for orchestration.  

This repository includes a Vagrant file that will set up a demo topology as depicted below.  The demo topology is identical to [Cumulus in the Cloud](https://cumulusnetworks.com/products/cumulus-in-the-cloud/).  

After bringing up the simulation and running the Ansible playbook installed on the out of band management server for you,  a BGP unnumbered underlay from the spines to the hosts, running FRR on the hosts is configured. The servers will have layer 3 connectivity to 2 leafs and the leafs are connected in a basic Clos topology to two spines.  NetQ will be installed on the network.  

Kubernetes is installed on the Ubuntu servers, with 2 deployments, apache1 and apache2.  Each deployment includes 5 instances of the container.   Server01 is the master node.

NetQ is used to view the container network.


Run the demo
--------------------


**Due to the topology and nature of kubernetes, this demo alone takes over 14G of memory - it cannot be run from a typical laptop.  Please allocate a server to run this demo.** 

**Step 1:  Install Vagrant, KVM, and libvirt on your simulation server.**

An Ansible playbook to install these technologies can be found at [Setting up a Ubuntu Server for Simulation](https://github.com/CumulusNetworks/ansible_snippets/tree/master/setup_simulation_server)  or Instructions can be found at [Vagrant and Libvirt with KVM](https://docs.cumulusnetworks.com/display/VX/Vagrant+and+Libvirt+with+KVM+or+QEMU) .  Choose Vagrant 2.1.1.  Be sure to install the vagrant-libvirt and vagrant-mutate plugins.

**Step 2:  Download the NetQ Telemetry Server VM.**

Download the NetQ TS VM from [Cumulus Website.](https://cumulusnetworks.com/downloads/#product=NetQ%20Virtual&hypervisor=Vagrant.)  You must be logged in to access.  Choose NetQ 1.3 with Vagrant hypervisor.  This will also become the oob-mgmt-server VM.

**Step 3:  Convert the vagrant NetQ TS to libvirt**

    vagrant mutate cumulus-netq-telemetry-server-amd64-1.3.0-vagrant.box libvirt

**Step 4:  Git clone this repository from your simulation server**


  cumulus@netq1:~$ git clone https://github.com/CumulusNetworks/cl_netq_demo_k8s
  cumulus@netq1:~$ cd cl_netq-demo-k8s



**Step 5:  Bring up the simulation**

As the oog-mgmt-server acts as DHCP server to the switches and hosts, bring those up first.  The --provision=libvirt only needs to be added the first time bringing up the topology on the simulation server.

    cumulus@netq1:~/cl_netq-demo-k8s$ vagrant up oob-mgmt-server oob-mgmt-switch --provision=libvirt
    
    cumulus@netq1:~/cl_netq-demo-k8s$ vagrant up spine01 spine02 leaf01 leaf02 leaf03 leaf04 server01 server02 server03 server04  --provision=libvirt

**Step 6:  Log into the simulation and run the kubernetes playbook**

    cumulus@netq1:~/cl_netq_demo_k8s$ vagrant ssh oob-mgmt-server

Password is *vagrant*

    cumulus@oob-mgmt-server:~$ cd k8s
    cumulus@oob-mgmt-server:~/k8s$ ansible-playbook setup.yaml

After the playbook completes, you can log into any node in the topology by sshing from the oob--mgmt-server.

Software in Use
------------------------
**Spines, Leafs**
      Cumulus VX 3.5.3

**On Servers:**
Ubuntu 16.04

**NetQ**
   Version 1.3

## Topology ##

This demo runs on a spine-leaf topology running BGP unnumbered with four attached hosts.  The oob-mgmt-server is connected to an out-of-band management network, along with all eth0 interfaces. 

![NetQ with K8S Demo Topology](https://github.com/CumulusNetworks/cl_netq_demo_k8s/blob/master/topology.png)

The following IP addresses are the loopback addresses on the hosts:
server01: 10.0.0.31  (master)
server02: 10.0.0.32
server03: 10.0.0.33
server04: 10.0.0.34

IPAM for data containers:  10.244.0.0/16
 
We run FRR on the host that advertises the servers loopback addresses and the container addresses (in the 10.244.X.X/26 domain) into the fabric.  FRR on the host also advertise the container addresses on each host into the fabric for container to container connectivity through the fabric.

We use K8S to deploy 2 services - apache1 and apache2, which are just httpd. Each service deploys 5 containers. We do not specify which servers they are deployed on, k8s decides that.




      cumulus@server01:~$ netq show kubernetes cluster
    
    Matching kube_cluster records:
    Master                   Cluster Name     Controller Status    Scheduler Status Nodes
    ------------------------ ---------------- -------------------- ---------------- --------------------
    server01:10.0.0.31       default          Healthy              Healthy          server01 server02 se                                                                                  rver03 server04

Next, look at the deployment instances

    cumulus@oob-mgmt-server:~$ netq show kubernetes deployment name apache1
    
    Matching kube_deployment records:
    Master                   Namespace       Name                 Replicas                           Ready Replicas Labels                         Last Changed
    ------------------------ --------------- -------------------- ---------------------------------- -------------- ------------------------------ ----------------
    server01:10.0.0.31       default         apache1              5                                  5              app:apache1                    16m:30.190s

Look at where they are deployed:     

         cumulus@oob-mgmt-server:~$ netq show kubernetes pod label apache2
    
    Matching kube_pod records:
    Master                   Namespace    Name                 IP               Node         Labels               Status   Containers               Last Changed
    ------------------------ ------------ -------------------- ---------------- ------------ -------------------- -------- ------------------------ ----------------
    server01:10.0.0.31       default      apache2-5b4c5d6dcc-8 10.244.6.4       server02     app:apache2          Running  apache2:c7f71b1b0b5b     16m:54.600s
                                          489n
    server01:10.0.0.31       default      apache2-5b4c5d6dcc-c 10.244.6.2       server02     app:apache2          Running  apache2:6a588b0e747e     16m:54.600s
                                          5dx9
    server01:10.0.0.31       default      apache2-5b4c5d6dcc-h 10.244.6.0       server02     app:apache2          Running  apache2:5f73ffc9d7c5     16m:54.600s
                                          2cvg
    server01:10.0.0.31       default      apache2-5b4c5d6dcc-n 10.244.6.1       server02     app:apache2          Running  apache2:9a4097c7c287     16m:54.600s
                                          56hj
    server01:10.0.0.31       default      apache2-5b4c5d6dcc-r 10.244.6.3       server02     app:apache2          Running  apache2:09404684319a     16m:54.600s
                                          5jss
    

Look at how a service is connected to the network

    
          cumulus@oob-mgmt-server:~$ netq show kubernetes deployment name apache1 connectivity
    apache1 -- apache1-86dd4b757f-lx9mt -- server03:eth1:eth1 -- swp1:swp1:leaf03
                                        -- server03:eth2:eth2 -- swp1:swp1:leaf04
            -- apache1-86dd4b757f-9n8cd -- server03:eth1:eth1 -- swp1:swp1:leaf03
                                        -- server03:eth2:eth2 -- swp1:swp1:leaf04
            -- apache1-86dd4b757f-jjl4h -- server02:eth1:eth1 -- swp2:swp2:leaf01
                                        -- server02:eth2:eth2 -- swp2:swp2:leaf02
            -- apache1-86dd4b757f-8prmx -- server02:eth1:eth1 -- swp2:swp2:leaf01
                                        -- server02:eth2:eth2 -- swp2:swp2:leaf02
            -- apache1-86dd4b757f-rnmvh -- server03:eth1:eth1 -- swp1:swp1:leaf03
                                        -- server03:eth2:eth2 -- swp1:swp1:leaf04

Drain a node from the master and view the changes:

    cumulus@server01:~$ kubectl drain server03 --delete-local-data --force --ignore-daemonsets
    node "server03" cordoned
    WARNING: Ignoring DaemonSet-managed pods: calico-node-wcptd, kube-proxy-2lvtw
    pod "apache1-86dd4b757f-rnmvh" evicted
    pod "apache1-86dd4b757f-9n8cd" evicted
    pod "apache1-86dd4b757f-lx9mt" evicted
    node "server03" drained

View the changes:

    cumulus@server01:~$ netq show kubernetes pod label apache1 changes
    
    Matching kube_pod records:
    Master                   Namespace    Name                 IP               Node         Labels           Status   Containers           Last Changed     DBState
    ------------------------ ------------ -------------------- ---------------- ------------ ---------------- -------- -------------------- ---------------- -------- ----------------
    server01:10.0.0.31       default      apache1-86dd4b757f-r 10.244.40.194    server03     app:apache1      Running  apache1:a677b2858a99 6m:21.262s       Del
                                          nmvh
    server01:10.0.0.31       default      apache1-86dd4b757f-l 10.244.40.193    server03     app:apache1      Running  apache1:eb7b88cd16e2 6m:21.264s       Del
                                          x9mt
    server01:10.0.0.31       default      apache1-86dd4b757f-9 10.244.40.192    server03     app:apache1      Running  apache1:cb0fa2a5574e 6m:21.259s       Del
                                          n8cd
    server01:10.0.0.31       default      apache1-86dd4b757f-x                  server04     app:apache1      Pending                       6m:21.370s       Add
                                          jdzt
    server01:10.0.0.31       default      apache1-86dd4b757f-x 10.244.135.130   server04     app:apache1      Running  apache1:b02399cc30d5 6m:6.742s        Add
                                          jdzt
    server01:10.0.0.31       default      apache1-86dd4b757f-x                  server04     app:apache1      Pending                       6m:21.370s       Add
                                          bb9z
    server01:10.0.0.31       default      apache1-86dd4b757f-x 10.244.135.129   server04     app:apache1      Running  apache1:83de610065aa 6m:6.749s        Add
                                          bb9z

What was it like 10 min ago?

    cumulus@server01:~$  netq show kubernetes pod label apache1 around 10m
    
    Matching kube_pod records:
    Master                   Namespace    Name                 IP               Node         Labels               Status   Containers               Last Changed
    ------------------------ ------------ -------------------- ---------------- ------------ -------------------- -------- ------------------------ ----------------
    server01:10.0.0.31       default      apache1-86dd4b757f-8 10.244.6.7       server02     app:apache1          Running  apache1:46f091b3f992     20m:1.842s
                                          prmx
    server01:10.0.0.31       default      apache1-86dd4b757f-9 10.244.40.192    server03     app:apache1          Running  apache1:cb0fa2a5574e     20m:2.842s
                                          n8cd
    server01:10.0.0.31       default      apache1-86dd4b757f-j 10.244.6.6       server02     app:apache1          Running  apache1:b303b10c5c6e     20m:2.842s
                                          jl4h
    server01:10.0.0.31       default      apache1-86dd4b757f-l 10.244.40.193    server03     app:apache1          Running  apache1:eb7b88cd16e2     20m:1.842s
                                          x9mt
    server01:10.0.0.31       default      apache1-86dd4b757f-r 10.244.40.194    server03     app:apache1          Running  apache1:a677b2858a99     20m:1.842s
                                          nmvh



Pipe to grep commands


    cumulus@oob-mgmt-server:~$ netq show kubernetes pod label apache1 changes|grep "server0[43]"
    server01:10.0.0.31       default      apache1-86dd4b757f-q                  server03     app:apache1      Pending                       1h:4m:0s         Del
    server01:10.0.0.31       default      apache1-86dd4b757f-p                  server03     app:apache1      Pending                       1h:4m:0s         Del
    server01:10.0.0.31       default      apache1-86dd4b757f-5                  server03     app:apache1      Pending                       1h:4m:0s         Del
    server01:10.0.0.31       default      apache1-86dd4b757f-q                  server03     app:apache1      Pending                       1h:17m:3s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-q 10.244.40.194    server03     app:apache1      Running  apache1:dd6ef0753c85 1h:16m:33s       Add
    server01:10.0.0.31       default      apache1-86dd4b757f-q                  server03     app:apache1      Pending                       1h:4m:16s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-p                  server03     app:apache1      Pending                       1h:17m:3s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-p 10.244.40.192    server03     app:apache1      Running  apache1:cd511dc99273 1h:16m:33s       Add
    server01:10.0.0.31       default      apache1-86dd4b757f-p                  server03     app:apache1      Pending                       1h:4m:16s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-k                  server04     app:apache1      Pending                       1h:4m:16s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-k 10.244.135.129   server04     app:apache1      Running  apache1:d1373243061e 1h:3m:45s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-d                  server04     app:apache1      Pending                       1h:4m:16s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-d 10.244.135.128   server04     app:apache1      Running  apache1:1b723b42fdbe 1h:3m:45s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-8                  server04     app:apache1      Pending                       1h:4m:16s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-8 10.244.135.130   server04     app:apache1      Running  apache1:28614cef0c09 1h:3m:45s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-5                  server03     app:apache1      Pending                       1h:17m:3s        Add
    server01:10.0.0.31       default      apache1-86dd4b757f-5 10.244.40.193    server03     app:apache1      Running  apache1:690b5e21b192 1h:16m:33s       Add
    server01:10.0.0.31       default      apache1-86dd4b757f-5                  server03     app:apache1      Pending                       1h:4m:16s        Add





