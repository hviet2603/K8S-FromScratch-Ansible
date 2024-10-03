# Build a K8s cluser v1.30.4 with Cilium network addon and HAProxy ingress controller

This repository stores Vagrantfiles and Ansible playbooks for building a K8s cluster v1.30.4 with Cilium as the network addon and HAProxy implementation for the ingress controller. The cluster composed of 1 master node and 2 worker nodes.

## Preparation

### Nodes configuration

`vagrant` is used to set up the nodes. Configuration for these VMs can be adapted at `vagrant/config.yaml`:

- Set `host_adapter` to an interface on your host, this adapter will be used as the bridged adapter to the VMs.

- You should generate an SSH keypair on your host and provide the path to the public key at `host_ssh_pubkey`.

- The static IP addresses for the VMs can also be changed, they should be in the same subnet. **Note:** when changing the IP address of the master node, the `master_ip` field in the config files in `host_vars` folder should be updated accordingly.

### Bringing up the nodes

The master and the workers can be started by running `vagrant up` in the folders `vagrant/master` and `vagrant/worker` respectively.

You may want to add an static IP address on your host that locates in the same subnet with the VMs, e.g. with `sudo ip a a dev eth0 192.168.0.100/24`.

The following entries should also be added to your `/etc/hosts` file:

```
192.168.0.2  local-k8s-master 
192.168.0.3  local-k8s-vworker1
192.168.0.4  local-k8s-vworker2
```

The IP addresses should be adapted according to your configuration. You can also install the `vagrant-hostsupdater` plugin to add these entries automatically on bringing up the VMs (`vagrant plugin install vagrant-hostsupdater`).

## Building the cluster

### Building the cluster using Ansible playbooks

Run the Ansible playbooks with the command `ansible-playbook ${playbook}`:

1. `01-kube_utils.yml`:

    -   Install the container runtime (`runc`, `containerd`, `cni`) and K8s utilities (`kubeadm`, `kubelet`, `kubectl`) on the nodes.
    
    -   Set up network related services and modules: enable IP forwarding and the module br_netfilter for the nodes.

2. `02-kube_master.yml`: Initialize the master node or control plane of the cluster.

3. `03-kube_joins.yml`: Join the worker nodes to the cluster.

4. `04-kube_network_addon.yml`: Install Cilium as the network addon for the cluster.

### Verifying the installation

Login to the master node as the root user.

Export the path to the admin config file with `export KUBECONFIG=/etc/kubernetes/admin.conf`.

Run `kubectl get pod -n kube-system`, all pods should now be in the the `Running` state.

## Installing the HAProxy ingress controller

[The Kubernetes ingress concept](https://kubernetes.io/docs/concepts/services-networking/ingress/).

### Installing the ingress controller

Run the command `ansible-playbook 05-kube_ingress_controller.yaml`. The manifest for the installation is modified from <https://github.com/haproxytech/kubernetes-ingress/blob/master/deploy/haproxy-ingress.yaml>.

### Testing the ingress controller

- Deploy an sample Nginx instance by running `ansible-playbook 06-kube_deploy_nginx_app.yaml`.

- Add the hostname `nginx.example.com` in association with the IP address from **any** of yours worker nodes in your `/etc/hosts` file:

```
192.168.0.3  nginx.example.com
```

- Visit `http://nginx.example.com:30080/` on your browser. If you see the Nginx welcome page, then the ingress controller is working as expected with the configured hostname :tada:.