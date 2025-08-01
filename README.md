# Build a Kubernetes cluster using K3s via Ansible

## Command Quick Reference

Test inventory SSH access with ping

```bash
ansible all -i inventory.yml -m ping
```

Run the playbook for installing `k3s`

```bash
ansible-playbook -K playbooks/site.yml -i inventory.yml
```

Run playbook for resetting the nodes, uninstalling `k3s`

```bash
ansible-playbook -K playbooks/reset.yml -i inventory.yml
```

Run playbook for upgrading `k3s`

```bash
ansible-playbook playbooks/upgrade.yml -i inventory.yml
```

Easily bring up a cluster on machines running:

- [X] Debian
- [X] Ubuntu
- [X] Raspberry Pi OS
- [X] RHEL Family (CentOS, Redhat, Rocky Linux...)
- [X] SUSE Family (SLES, OpenSUSE Leap, Tumbleweed...)
- [X] ArchLinux

on processor architectures:

- [X] x64
- [X] arm64
- [X] armhf

## System requirements

The control node **must** have Ansible 8.0+ (ansible-core 2.15+)

All managed nodes in inventory must have:

- Passwordless SSH access
- Root access (or a user with equivalent permissions)

It is also recommended that all managed nodes disable firewalls and swap. See [K3s Requirements](https://docs.k3s.io/installation/requirements) for more information.

## Installation

### With ansible-galaxy

`k3s-ansible` is a Ansible collection and can be installed with the `ansible-galaxy` command:

```console
ansible-galaxy collection install git+https://github.com/k3s-io/k3s-ansible.git
```

### From source

Alternatively to an installation with `ansible-galaxy`, the `k3s-ansible` repository can simply be cloned from github:

```console
git clone https://github.com/k3s-io/k3s-ansible.git
cd k3s-ansible
```

## Usage

First copy the sample inventory to `inventory.yml`.

```bash
cp inventory-sample.yml inventory.yml
```

Test inventory SSH access with ping

```bash
ansible all -i inventory.yml -m ping
```

If you have installed `k3s-ansible` with ansible-galaxy, you can grab the [inventory-sample.yml](./inventory-sample.yml) from github.

Second edit the inventory file to match your cluster setup. For example:

```bash
k3s_cluster:
  children:
    server:
      hosts:
        hostname: address
    agent:
      hosts:
        hostname: address
```

If needed, you can also edit `vars` section at the bottom to match your environment.

If multiple hosts are in the server group the playbook will automatically setup k3s in HA mode with embedded etcd.
An odd number of server nodes is required (3,5,7). Read the [official documentation](https://docs.k3s.io/datastore/ha-embedded) for more information.

Setting up a loadbalancer or VIP beforehand to use as the API endpoint is possible but not covered here.

Start provisioning of the cluster using one of the following commands. The command to be used depends on whether you installed `k3s-ansible` with `ansible-galaxy` or if you run the playbook from within the cloned git repository:

*Installed with ansible-galaxy*

```bash
ansible-playbook k3s.orchestration.site -i inventory.yml
```

*Running the playbook from inside the repository*

```bash
ansible-playbook -K playbooks/site.yml -i inventory.yml
```

### Using an external database

If an external database is preferred, this can be achieved by passing the `--datastore-endpoint` as an extra server argument as well as setting the `use_external_database` flag to true.

```bash
k3s_cluster:
  children:
    server:
      hosts:
        hostname1: address1
    agent:
      hosts:
        hostname2: address2

  vars:
    use_external_database: true
    extra_server_args: "--datastore-endpoint=postgres://username:password@hostname:port/database-name"
```

The `use_external_database` flag is required when more than one server is defined, as otherwise an embedded etcd cluster will be created instead.

The format of the datastore-endpoint parameter is dependent upon the datastore backend, please visit the [K3s datastore endpoint format](https://docs.k3s.io/datastore#datastore-endpoint-format-and-functionality) for details on the format and supported datastores.

## Upgrading

A playbook is provided to upgrade K3s on all nodes in the cluster. To use it, update `k3s_version` with the desired version in `inventory.yml` and run one of the following commands. Again, the syntax is slightly different depending on whether you installed `k3s-ansible` with `ansible-galaxy` or if you run the playbook from within the cloned git repository:

*Installed with ansible-galaxy*

```bash
ansible-playbook k3s.orchestration.upgrade -i inventory.yml
```

*Running the playbook from inside the repository*

```bash
ansible-playbook playbooks/upgrade.yml -i inventory.yml
```

## Airgap Install

Airgap installation is supported via the `airgap_dir` variable. This variable should be set to the path of a directory containing the K3s binary and images. The release artifacts can be downloaded from the [K3s Releases](https://github.com/k3s-io/k3s/releases). You must download the appropriate images for you architecture (any of the compression formats will work).

An example folder for an x86_64 cluster:

```bash
$ ls ./playbooks/my-airgap/
total 248M
-rwxr-xr-x 1 $USER $USER  58M Nov 14 11:28 k3s
-rw-r--r-- 1 $USER $USER 190M Nov 14 11:30 k3s-airgap-images-amd64.tar.gz

$ cat inventory.yml
...
airgap_dir: ./my-airgap # Paths are relative to the playbooks directory
```

Additionally, if deploying on a OS with SELinux, you will also need to download the latest [k3s-selinux RPM](https://github.com/k3s-io/k3s-selinux/releases/latest) and place it in the airgap folder.

It is assumed that the control node has access to the internet. The playbook will automatically download the k3s install script on the control node, and then distribute all three artifacts to the managed nodes.

## Kubeconfig

After successful bringup, the kubeconfig of the cluster is copied to the control node  and merged with `~/.kube/config` under the `k3s-ansible` context.
Assuming you have [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) installed, you can confirm access to your **Kubernetes** cluster with the following:

```bash
kubectl config use-context k3s-ansible
kubectl get nodes
```

If you wish for your kubeconfig to be copied elsewhere and not merged, you can set the `kubeconfig` variable in `inventory.yml` to the desired path.

## ArgoCD Access

After successful installation, you can access ArgoCD through the load balancer endpoint. The admin password can be retrieved using:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

The load balancer endpoint can be found using:

```bash
kubectl get svc -n argocd argocd-server
```

The default username is `admin`. Use the password from the secret to log in to the ArgoCD UI.

## Resetting the cluster

A playbook is provided to reset the cluster. To use it, run one of the following commands. Again, the syntax is slightly different depending on whether you installed `k3s-ansible` with `ansible-galaxy` or if you run the playbook from within the cloned git repository:

*Installed with ansible-galaxy*

```bash
ansible-playbook k3s.orchestration.reset -i inventory.yml
```

*Running the playbook from inside the repository*

```bash
ansible-playbook -K playbooks/reset.yml -i inventory.yml
```

## Local Testing

A Vagrantfile is provided that provision a 5 nodes cluster using Vagrant (LibVirt or Virtualbox as provider). To use it:

```bash
vagrant up
```

By default, each node is given 2 cores and 2GB of RAM and runs Ubuntu 20.04. You can customize these settings by editing the `Vagrantfile`.

## Need More Features?

This project is intended to provide a "vanilla" K3s install. If you need more features, such as:

- Private Registry
- Advanced Storage (Longhorn, Ceph, etc)
- External Database
- External Load Balancer or VIP
- Alternative CNIs

See these other projects:

- <https://github.com/PyratLabs/ansible-role-k3s>
- <https://github.com/techno-tim/k3s-ansible>
- <https://github.com/jon-stumpf/k3s-ansible>
- <https://github.com/alexellis/k3sup>
- <https://github.com/axivo/k3s-cluster>

