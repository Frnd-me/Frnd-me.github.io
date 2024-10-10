---
layout: post
title: Provisioning a Local Kubernetes Cluster with Ansible and Vagrant
description: This post explains how to set up a local Kubernetes cluster using Ansible and Vagrant.
summary: A guide to provisioning a local Kubernetes cluster using Ansible and Vagrant.
tags: [kubernetes, ansible, vagrant]
---

In this post, I'll walk you through setting up a local Kubernetes cluster using two powerful tools: Ansible and Vagrant. Both tools are widely used in DevOps workflows to automate infrastructure management; each serves a unique purpose in this setup.

# Vagrant

Vagrant is a tool for managing virtual environments efficiently. It wraps virtualization providers like VirtualBox and others, making creating and managing consistent development environments easy. By writing a configuration file (called a Vagrantfile), you can define the resources you need, like virtual machines (VMs), for each node of the Kubernetes cluster.

For a local Kubernetes cluster, Vagrant's key advantage is that it simplifies creating and managing multiple VMs (such as the master and worker nodes). It ensures that each VM is provisioned with consistent specifications, like the correct amount of memory, CPU, and networking configuration.

# Ansible

Ansible is an automation tool focusing on configuration management, application deployment, and infrastructure orchestration. It configures and manages machines using playbooks — simple YAML files describing tasks. What makes Ansible particularly useful is that it's agentless. Unlike some other configuration management tools, Ansible doesn't require software agents to be installed on the target machines. Instead, it uses SSH to execute tasks remotely.

In setting up our local Kubernetes cluster, Ansible automates the installation of required software (like Docker, kubeadm, kubectl, and Kubernetes networking components) across all nodes in the cluster. Rather than manually installing and configuring each node, you can define these steps in a playbook, which ensures that the entire cluster is configured uniformly.

# Why use Vagrant and Ansible together?

Combining Vagrant and Ansible brings the best of both worlds: Vagrant is great for managing virtual machines. At the same time, Ansible is ideal for automating the configuration of those VMs. Here's how they complement each other:
Vagrant creates the VMs (master and worker nodes) that act as your Kubernetes cluster.
Ansible then configures those VMs, installing Kubernetes and setting up the cluster automatically.

# Why not Minikube?

One of the main limitations of Minikube is that it runs Kubernetes in a single-node configuration (both master and worker processes on the same VM). While this is fine for basic testing, it doesn't fully simulate a real-world production environment. It typically involves multiple nodes (one or more master nodes and several worker nodes).

Using Vagrant and Ansible, you can create an actual multi-node cluster with separate VMs for master and worker nodes. This provides a closer approximation to how Kubernetes runs in production.

# Vagrantfile

A Vagrantfile serves as a configuration file for Vagrant, facilitating the creation and management of virtualized development environments. In this specific Vagrantfile, a Kubernetes cluster is established with one master node and two worker nodes, all running on Ubuntu 22.04 using a base box from Bento. Each virtual machine is configured with a unique private IP address alongside allocated CPU and memory resources. The master node is set up with port forwarding to enable Kubernetes API access on port 6443. Ansible is employed directly from within Vagrant for provisioning, executing a predefined playbook to configure Kubernetes on each VM using the Ansible Provisioner.

Let's proceed with the section of the Vagrantfile that configures the master:

```ruby
Vagrant.configure("2") do |config|

    # Configuration for the "master" VM
    config.vm.define "master" do |master|

        # Define a static IP for the master VM
        ip = "10.10.2.10"

        # Set the base box for the master VM
        master.vm.box = LINUX_BASE_BOX

        # Set the hostname for the master VM
        master.vm.hostname = "master"

        # Configure a private network with the defined IP
        master.vm.network "private_network", ip: ip

        # Forward port 6443 from the guest (VM) to the host machine, for Kubernetes API access
        master.vm.network "forwarded_port", guest: 6443, host: 6443

        # Configure the master VM's provider (VirtualBox in this case)
        master.vm.provider "virtualbox" do |virtualbox|

            # Customize the VM settings
            virtualbox.customize ["modifyvm", :id, "--cableconnected1", "on", "--audio", "none"]

            # Set the memory to 2048 MB (2 GB)
            virtualbox.memory = "2048"

            # Allocate 2 CPUs to the master VM
            virtualbox.cpus = "2"
        end

        # Provision the master VM using Ansible to run the specified playbook
        master.vm.provision "ansible" do |ansible|

            # Specify the Ansible playbook file for Kubernetes setup
            ansible.playbook = "./ansible/kubernetes.yml"

            # Pass the master node's IP address as an extra variable to the playbook
            ansible.extra_vars = {
                node_ip: ip,
            }
        end
    end
```

Similarly, two additional worked nodes are set up:

```ruby
    # Loop to configure 2 node VMs (node1 and node2)
    1.upto(2) do |n|

        # Configuration for each node VM
        config.vm.define "node#{n}" do |node|

            ip = "10.10.2.%d" % [20 + n]

            node.vm.box = LINUX_BASE_BOX
            node.vm.hostname = "node#{n}"
            node.vm.network "private_network", ip: ip

            node.vm.provider "virtualbox" do |virtualbox|

                virtualbox.customize ["modifyvm", :id, "--cableconnected1", "on", "--audio", "none"]
                virtualbox.memory = "2048"
                virtualbox.cpus = "2"
            end

            node.vm.provision "ansible" do |ansible|

                ansible.playbook = "./ansible/kubernetes.yml"
                ansible.extra_vars = {
                    node_ip: ip,
                }
            end
        end
    end
end
```

The configuration file is saved in a file named _Vagrantfile_.
