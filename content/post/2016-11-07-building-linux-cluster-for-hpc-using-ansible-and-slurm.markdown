---
title: Building Linux Cluster for HPC using Ansible and SLURM
date: 2016-11-07T06:06:24+07:00
excerpt: ""
featured_image: images/sample-image-3.jpg
featured_image_class: contain bg-center
header:
  overlay_image: sample-image-3.jpg
  caption: "Photo credit: [**WeGraphics**](http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/)"
---

## What is a Linux Cluster?

Linux cluster is a group of computers running Linux which is interconnected in a local network so they can communicate with each other. Linux cluster is often used in high performance computing to do complex calculation of scientific problems because they can share the calculation load across all computers in the cluster. One example of the technology used to share the calculation load to all computers in the cluster is MPI or Message Passing Interface. There are many implementations of MPI, for example OpenMPI, MPICH, and Intel MPI.

## High Performance Computing

High performance computing is a method using parallel processing to run advanced computer applications more efficiently, reliably, and quickly and usually refers to the practice of aggregating computing power of several computers to get higher performance than one could get from a single computer to solve complicated problems in science, engineering or business.

## Waste of Computing Power

A typical datacenter which runs internet service like web and email hosting usually doesn't consume much of its raw computing power and the unused processor's cores become a waste of electricity. One idea to utilize the unused computing power is to create a simple cluster underneath application which serve the usual internet service to run some parallel application which can be used to solve complicated but parallelizable problems. Of course the use of the cluster must be strictly moderated so it won't disturb the operational of main internet service.

## High Performance Computer Cluster

A high performance computer is a computer with special specification which provides it with a high computational power in form of processor's technology, processor's number of cores, size and bandwidth of memory, large capacity harddisks, very fast storage and networking devices, etc. A computer which used as a server in data center usually has a specification close to the specification of a high performance computer. A group of high performance computer can be combined as a cluster to calculate complex problems using special computer applications or programs. In Chemical Computation, example of such programs is QuantumESPRESSO, which is suitable to investigate a crystalline structure, and GAMESS, which is suitable to investigate a molecular structure.

### Ansible

[Ansible](https://www.ansible.com/) is a configuration management tool which commonly used to manage multiple servers simultaneously in DevOps practice. But, its functionality can be used not only to build a cloud infrastructure but also to build a high performance computer cluster. Configuration of compute nodes can be done automatically and simultaneously from a single master node thus removing needs to log in one-by-one to each compute node. Another configuration management tool which already popular in HPC practice is C3.

### SLURM (Simple Linux Utility for Resource Management)

[SLURM](slurm.schedmd.com/) is a cluster management tool and job scheduling system for linux clusters. In other words, user can submit calculation job or manage cluster resources allocation using SLURM. SLURM has several main functions: resources allocation, running job management, and queue management, and more functionalities can be added using plugin. SLURM is an open-source tool which highly scalable and fault-tolerant and requires no kernel modification for its operation.

![SLURM and Modules](/images/linux-cluster-1.png)
