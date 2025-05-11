# Kubernetes Vagrant Config

> [!CAUTION]
> This repository was created as a learning exercise.

This repository contains a Vagrantfile to describe a set of virtual machines to create a small Kubernetes cluster.

This was developed against the VMware provider for VMware Fusion, but may also work for VMware Workstation. On Mac, the dependencies can be installed with:

    brew install vagrant vagrant-vmware-utility vmware-fusion
    vagrant plugin install vagrant-vmware-desktop

To start the cluster, simply run:

    vagrant up

To control the cluster:

    vagrant ssh
    vagrant ssh -- -- kubectl get nodes

Ports can be forwarded from the host to a pod or service with:

    vagrant ssh \
      -- -L <host port>:localhost:<guest port> \
      -- kubectl port-forward pod/podname <guest port>:<pod port>


### Networking

The nodes share a virtual network. Due to a [limitation since macOS 11 Big Sur][big-sur-bug], it is not possible to assign a static IP to each node. As a workaround, dynamic IPs are written to a file which is synchronized across the virtual machines. **Please note that this is not secured against tampering by any compromised node.**

[big-sur-bug]: https://developer.hashicorp.com/vagrant/docs/providers/vmware/known-issues#big-sur-related-issues
