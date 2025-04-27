---
title: Upgrade OpenStack Mitaka to OpenStack Newton using RDO
date: 2017-01-06T18:42:29+07:00
excerpt: "My OpenStack development environment is build using RDO's packstack utility and consists of three nodes: one controller, network, and compute node, and two compute nodes. Each node which runs the OpenStack services actually a virtual machine (VM) running above multiple type of hypervisors and operating systems."
featured_image: images/openstack-newton.png
featured_image_class: contain bg-center
header:
  overlay_image: openstack-newton.png
  overlay_filter: 0.7
---

My OpenStack development environment is build using RDO's **packstack** utility and consists of three nodes: one controller, network, and compute node, and two compute nodes. RDO website offers two scenario for [upgrading](https://www.rdoproject.org/install/upgrading-rdo/) OpenStack Mitaka to OpenStack Newton. The first scenario involves taking down all of the OpenStack services at once and will not bring them back up until the upgrade process is completed. The second scenario upgrades each OpenStack service one by one to avoid downtime by performing rolling upgrades of the compute hosts, taking advantage of the fact that nova-compute from Mitaka can communicate with a Newton control plane.

My development environment is simple, so I choose the first scenario to upgrade my OpenStack installation. Each node which runs the OpenStack services actually a virtual machine (VM) running above multiple type of hypervisors and operating systems. The controller node is a CentOS 7 VM running above KVM in CentOS 7 host, the first compute node is a CentOS 7 VM running above KVM in ArchLinux host, and the second compute node is a CentOS 7 VM running above BHYVE in FreeBSD 10 host. The second compute node doesn't have capabilities to do virtualization so in this compute node I use [Docker](https://www.docker.com/) as compute driver using a OpenStack project called [Nova-Docker](https://github.com/openstack/nova-docker).

## The Upgrades

The upgrade process is running smoothly at first

### Disabling all OpenStack Services

Install **openstack-utils** on all standard nodes to manage OpenStack services with **openstack-service** command

        sudo yum install openstack-utils

Then, disable all OpenStack services on all standard nodes

        sudo openstack-service stop

### Performing a Package Upgrade

Install the Newton release repository

        sudo yum install -y centos-release-openstack-newton

Disable previous release repositories, for example

        sudo yum-config-manager --disable centos-release-openstack-mitaka

Then update the packages

        sudo yum update

Wait until the package upgrade is completed.
Review the resulting configuration files. The upgraded packages will have installed .rpmnew files appropriate to the Newton version of the service. New versions of OpenStack services may deprecate certain configuration options. You should also review your OpenStack logs for any deprecation warnings, because these may cause problems during future upgrades. For more information on the new, updated and deprecated configuration options for each service, see Configuration Reference available from <http://docs.openstack.org/newton/config-reference>.
Perform the package upgrade on each node in your environment.

### Performing Synchronization of all Databases

After we upgrade the packages, then we have to upgrade the database of each service.
Flush expired tokens in the Identity service to decrease the time required to synchronize the database

        sudo keystone-manage token_flush

Upgrade the database schema for each service that uses the database. Run the following commands on the node hosting the service’s database.

Table 1. Commands to Synchronize OpenStack Service Databases

| Service       | Project name | Command                                                        |
| :------------ | :----------- | :------------------------------------------------------------- |
| Identity      | keystone     | `# su -s /bin/sh -c "keystone-manage db_sync" keystone`        |
| Image Service | glance       | `# su -s /bin/sh -c "glance-manage db_sync" glance`            |
| Block Storage | cinder       | `# su -s /bin/sh -c "cinder-manage db sync" cinder`            |
| Orchestration | heat         | `# su -s /bin/sh -c "heat-manage db_sync" heat`                |
| Compute       | nova         | `# su -s /bin/sh -c "nova-manage api_db sync" nova`            |
|               |              | `# su -s /bin/sh -c "nova-manage db sync" nova`                |
| Telemetry     | ceilometer   | `# ceilometer-dbsync`                                          |
| Networking    | neutron      | `# su -s /bin/sh -c "neutron-db-manage upgrade heads" neutron` |

### Enabling all OpenStack Services

The final step enables the OpenStack services on the node. Restart all OpenStack services:

        sudo openstack-service start

## The Problems

Like usual, we ran into problems everytime we upgrade an environment.

### Resolving the configuration files differences is too much of works and I am too lazy to do it

I end up using **packstack** utility once again to re-configure my development environment. First, we generate a template for the new packstack answer file

        sudo packstack --gen-answer-file=ANSWER-FILE-NEWTON

Take your old answer file and change the passwords fields and others configuration option to suit your needs. I also upgrade the authentication schema to use the **v3** version. There are also another problem which is I cannot use my domain name for declaring the nodes, so I have to use IP addresses. Then, run the **packstack** utility.

        sudo packstack --answer-file=ANSWER-FILE-NEWTON

After the **packstack** utility finish successfully, reboot the OpenStack nodes.

        sudo reboot

After all the nodes have started again, try to access your OpenStack installation using your old credentials.

### The Console utilty in Instances module is not working

Using IP addresses to declare the nodes is having bad effect to the Nova configuration, I have to change novncproxy*base_url options*/etc/nova/nova.conf\_ in all KVM based compute nodes from using the controller IP address to use the controller domain name

        /etc/nova/nova.conf:
        ...
        #novncproxy_base_url=http://10.0.1.11:6080/vnc_auto.html
        novncproxy_base_url=http://controller:6080/vnc_auto.html

Restart the OpenStack services on all nodes

### The Nova-Docker project has been abandoned for OpenStack Newton

Sadly, the nova-docker project has been abandoned and there is no way I can use my second compute node to run docker instances on OpenStack. But all hope is not lost! I managed to hack the nova-docker installation to at least run my existing docker instance on OpenStack.

Get the latest version of nova-docker project in Github

        git clone https://github.com/openstack/nova-docker.git

Install the nova-docker

        cd nova-docker
        sudo python setup.py install

Install the **python-docker-py** package

        sudo yum install python-docker-py

Change the Nova configuration in */etc/nova/nova.conf*

        /etc/nova/nova.conf:
        ...
        #compute_driver=libvirt.LibvirtDriver
        compute_driver=novadocker.virt.docker.DockerDriver

Add filter file */etc/nova/rootwrap.d/docker.filters* writable only by root

        /etc/nova/rootwrap.d/docker.filters:
        ...
        # nova-rootwrap command filters for setting up network in the docker driver
        # This file should be owned by (and only-writeable by) the root user

        [Filters]
        # nova/virt/docker/driver.py: 'ln', '-sf', '/var/run/netns/.*'
        ln: CommandFilter, /bin/ln, root

**The Hack**. Create a symbolic link for the Nova compute service to pick the nova-docker module

        sudo ln -s /usr/lib/python2.7/site-packages/novadocker /usr/lib/python2.7/site-packages/nova/virt/novadocker

Edit file */usr/lib/python2.7/site-packages/novadocker/virt/docker/vifs.py* to accomodate deprecated configuration in OpenStack Newton

        /usr/lib/python2.7/site-packages/novadocker/virt/docker/vifs.py
        ...
        #CONF.import_opt('network_device_mtu', 'nova.objects.network')
        ...
        #mtu = CONF.network_device_mtu
        mtu = 1450

Restart the **openstack-nova-compute** service on the second node.

## References

1. [Upgrading from Mitaka to Newton: Overview](https://www.rdoproject.org/install/upgrading-rdo/)
