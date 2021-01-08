# Deploying Kubernetes Cluster on Baremetal with Ceph

This repository shows how to deploy k8s cluster using [rancher rke](https://rancher.com/docs/rke/latest/en/) tool on three baremetal nodes on Hetzner. The Hetzner nodes are provisioned using ansible playbooks. k8s installation is done with Rancher RKE tool. Additionally, we deploy a ceph cluster to enable automatic persistent volume provisioning in the cluster.

- [Deploying Kubernetes Cluster on Baremetal with Ceph](#deploying-kubernetes-cluster-on-baremetal-with-ceph)
  - [Necessary Tools](#necessary-tools)
  - [Cluster overview](#cluster-overview)
  - [Getting started](#getting-started)
  - [Provision the Virtual Machines](#provision-the-virtual-machines)
  - [Install Kubernetes Cluster](#install-kubernetes-cluster)
  - [Configurating DNS and Cert Manager for the Kubernetes Cluster](#configurating-dns-and-cert-manager-for-the-kubernetes-cluster)

## Necessary Tools

The following tools need to be installed on your machine:

* [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for creating and provisioning the virtual machines
* [hcloud cli](https://github.com/hetznercloud/cli) for working with Hetzner Cloud - download binary from Releases and put it on your PATH
* [jq](https://stedolan.github.io/jq/download/) for parsing JSON output from e.g. ansible
* [RKE](https://rancher.com/docs/rke/latest/en/installation/) for provisioning k8s cluster
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for talking to k8s cluster
* [helm](https://helm.sh/docs/intro/install/) for deploying apps on k8s cluster

## Cluster overview

In this scenario, we deploy 3 nodes cluster. Each node has 8 GiB RAM and 2 vCPUs. Additionally, the nodes will have extra volumes 10 GiB each. The total price for running such a cluster is equals to €5.83\*3 + €0.48\*3 = €18.93 per month. For running this cluster for several hours, the price will be well under €1.

## Getting started

To get started you will need a [Hetzner cloud account](https://www.hetzner.com/). After you have an account there, create a project and generate an API token for it (it's in Security --> API TOKENS). If necessary, consult [Official Hetzner Documentation](https://docs.hetzner.cloud/). Export token the environment:

```
export HETZNER_API_TOKEN=yourapitoken
```

Add your SSH key to the Hetzner cloud:

```
hcloud context create devops
# Paste your API token
hcloud ssh-key create --public-key-from-file ~/.ssh/id_rsa.pub --name devops
hcloud ssh-key list
```

Create the virtual machines:

```
ansible-playbook create_vms.yml
```

Setup Hetzner dynamic inventory:

```
sudo -E su
mkdir -p /etc/ansible
cat <<EOF > /etc/ansible/hosts.hcloud.yml
plugin: hcloud
token: ${HETZNER_API_TOKEN}
groups:
  devops: true
  hetzner: true
EOF
exit

ansible-galaxy collection install hetzner.hcloud
pip install hcloud
export ANSIBLE_INVENTORY=/etc/ansible/hosts.hcloud.yml
```

List the created VMs:

```
ansible-inventory --list
```

## Provision the Virtual Machines

Install necessary ansible packages and provision the virtual machines:

```
ansible-galaxy collection install community.general

export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -l devops provision_vm.yml
```

Validate provisioning:

```
export K8S1_IP=$(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
ssh root@${K8S1_IP} docker run --rm hello-world
```

## Install Kubernetes Cluster

Create rke configuration:

```
cat <<EOF > cluster.yml
ssh_agent_auth: true
cluster_name: k8s-hetzner
name: k8s-hetzner
enable_cluster_alerting: true
enable_cluster_monitoring: true
ignore_docker_version: true

nodes:
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
    hostname_override: k8s-1
    user: root
    labels:
      worker: yes
      location: fsn
    role: [controlplane, worker, etcd]
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-2"].ipv4' | tr -d '"')
    hostname_override: k8s-2
    user: root
    labels:
      worker: yes
      location: fsn
    role: [controlplane, worker, etcd]
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-3"].ipv4' | tr -d '"')
    hostname_override: k8s-3
    user: root
    labels:
      worker: yes
      location: fsn
    role: [controlplane, worker, etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 30h
  kube-controller:
    extra_args:
      terminated-pod-gc-threshold: 100
  kubelet:
    extra_args:
      max-pods: 250

ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"

monitoring:
  provider: "metrics-server"
EOF
```

Provision kubernetes cluster:

```
ssh-add ~/.ssh/id_rsa
rke up
```

Validate kubernetes cluster:

```
export KUBECONFIG=kube_config_cluster.yml
alias k=kubectl

k get no

cat <<EOF | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    app: busybox1
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
EOF

k get po -o wide
k delete po busybox1
```

## Configurating DNS and Cert Manager for the Kubernetes Cluster

Setup a wildcard A entry for all three IP addresses of the k8s cluster in your DNS provider:

```
for i in {1..3}; do bash -c "ansible-inventory --list | jq '._meta.hostvars' | jq '.[\"k8s-$i\"].ipv4' | tr -d '\"'" ; done
```

Install cert-manager:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml

# wait until deployment is done
kubectl -n cert-manager rollout status deploy/cert-manager
```

Install letsencrypt issuers for staging and prod:

```
export EMAIL=youremailaddress@example.com

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${EMAIL}
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ${EMAIL}
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```