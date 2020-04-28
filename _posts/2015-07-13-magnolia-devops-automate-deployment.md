---
layout: post
title: "Magnolia DevOps Series - Automate your deployment on multiple hosts"
date: 2015-07-13 00:00:00 -0000
---
In my last experiment with Docker I used [Docker Compose](https://github.com/docker/fig) to automate the creation of several Magnolia instances. 

The Docker Compose configuration file is neat and easy to understand but quite limited for a production usage:

* It does not support multiple hosts and provisioning <sup><a href="#fn1" id="ref1">1</a></sup>.
* Constrained orchestration features such as [blue/green](http://martinfowler.com/bliki/BlueGreenDeployment.html) deployment or database backup/restore require additional tools.

A different approach consists in using a more expressive orchestration tool such as [Puppet](https://puppetlabs.com/) or [Chef](https://www.chef.io/chef/). I choose [Ansible](http://www.ansible.com/home) for its friendly YAML configuration files and the usage of SSH instead of a proprietary agent.

### The Inventory
Ansible uses an inventory to list the components of your infrastructure. The provisioning can be static or dynamic (your cloud infrastructure feeds the Ansible's inventory through a specific connector).

Here is the inventory that I am using in this example:
```prettyprint lang-bash
[all:vars]
private_net_iface=eth1

# Deploy a Magnolia public instance on each host with the specified amount of nodes in the cluster
[magnolia-publics]
instance1.local cluster_node_count=1 author="instance3.local" port="9000"
instance2.local cluster_node_count=1 author="instance3.local" port="9000"

# Deploy a Magnolia author instance (clustering not availble on authors)
[magnolia-authors]
instance3.local port="9000"

[lbservers]
instance1.local
```
The systems are grouped into 3 categories, called _groups_ in Ansible:

* ```magnolia-publics```: Deploy on each host a cluster of Magnolia public instances with its shared database and the specified amount of nodes in the cluster. The cluster is automatically registered as a publisher in the specified author instance.
* ```magnolia-authors```: Deploy on each host a single Magnolia author instance with its database.
* ```lbservers```: The load balancer used to dispatch the traffic to the public instances.

The infrastructure can be configured through the following host variables in the inventory file:

* ```cluster_node_count```: Amount of nodes in the Magnolia cluster. A cluster must be initialized with only one node. Upscalling can be only done in a subsequent execution of the playbook.
* ```port```: Exposed HTTP port on the host of the Magnolia instance. If several nodes are present in the cluster, the port number is automatically incremented.
* ```author```: Name of the host where the author instance is deployed.

The statement ```[all:vars]``` contains global variables valid for all hosts. I am using ```net_iface``` to specify the name of the network interface exposed on the host. This value varies from one provider to another.

Describing the infrastructure in one single YAML file brings a lot of flexibility. Adding a new host, or increasing the amount of instances in an existing cluster is a simple modification that can be easily tracked in a source code control such as GIT, and tested in a preproduction environment before rolling the change in production.

### Deploy the site locally
The deployment script can be tested locally using the Vagrant provisioning available in the project.

You must install Vagrant to provision the virtual machines. If you don't have it on your computer, download a copy from their website [vagrant](http://www.vagrantup.com/downloads) and install it.

Download the source code of the project from git:
```
git clone https://github.com/nicolasbarbe/magnolia-ansible.git
```


Execute this command from the directory of the project to create the hosts:
```
vagrant up
```
It may ask for a password since it requires root privileges to declare the virtual machine inside ```/etc/host```.

You can check that the virtual machines are properly created running the command ```vagrant status``` which must return the output:

```
Current machine states:

instance1.local           running (virtualbox)
instance2.local           running (virtualbox)
instance3.local           running (virtualbox)
```
Now you are ready to deploy the site, simply execute this command from the project:
```
ansible-playbook -i inventories/local site.yml
```
Ansible will prepare the virtual machines and deploy the Magnolia instances with their database and the load-balancer. The public website is available through the load-balancer at this address: http://192.168.33.10. The author instance can be reach at this address: http://192.168.33.12:9000

### Deploy the site in production
The process to deploy the site in production is quite similar:

1. Create a few hosts on your cloud infrastructure and makes sure that the SSH port is opened. 
2. Create a new file called production in the inventories and configure it using the IP provided by your cloud infrastructure. Do not forget to update the name of the network interface.
3. Execute the playbook:
```
ansible-playbook -i inventories/production site.yml
```
For security reasons, I recommend to execute the playbook directly in your infrastructure on a dedicated host using a specific private network and without using any public IP nor exposing publically the SSH ports.

### Horizontal scaling
The infrastructure can be scaled out either by adding a new host under the `magnolia-publics` group or increasing the amount of nodes in the cluster of an existing host.

The first approach requires data synchronization between the existing public instances and the new one. The easiest way is to clone the docker container of an existing database and install it on the new host (including the JCR repository).

The second approach does not require data synchronization since the database is shared accross all the instances. The current implementation with ansible is however limited since the cluster is deployed on the same host.

*Note: The project used in the tutorial is based on the Community Edition which is limited to one subscriber. If you want to have more than one public instance, you must switch this project to the Enterprise Edition.*

### Some extra features
This example comes with two extra scripts to illustrate the flexibility of the combination of Ansible and Docker:

- list_apps: List the applications running in your infrastructure.
- clean_all: Remove all the applications and Docker images from the hosts.

The source code of this article can be found on [github](https://github.com/nicolasbarbe/magnolia-ansible). 

Finally, do not hestitate to use the comments below to share your own experience especially if you have followed a different approach!

<sup id="fn1">1. Docker is actively working on the integration of Compose and Swarm. Together with Docker Machine, this will bring a lot of new interesting possibilities to provision and manage a cluster of Docker applications.<a href="#ref1">↩</a></sup>
