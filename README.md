# Deploying Kubernetes Cluster on Baremetal with Ceph

This repository shows how to deploy k8s cluster using [rancher rke](https://rancher.com/docs/rke/latest/en/) tool on three baremetal nodes on Hetzner. The Hetzner nodes are provisioned using ansible playbooks. k8s installation is done with Rancher RKE tool. Additionally, we deploy a ceph cluster to enable automatic persistent volume provisioning in the cluster.

- [Deploying Kubernetes Cluster on Baremetal with Ceph](#deploying-kubernetes-cluster-on-baremetal-with-ceph)
  - [Necessary Tools](#necessary-tools)
  - [Cluster overview](#cluster-overview)
  - [Getting started](#getting-started)
  - [Provision the Virtual Machines](#provision-the-virtual-machines)
  - [Install Kubernetes Cluster](#install-kubernetes-cluster)
  - [Configurating DNS for the Kubernetes Cluster](#configurating-dns-for-the-kubernetes-cluster)
  - [Configuring Cert Manager for the Kubernetes Cluster](#configuring-cert-manager-for-the-kubernetes-cluster)
  - [Configuring Rancher UI for the Kubernetes Cluster](#configuring-rancher-ui-for-the-kubernetes-cluster)
  - [Troubleshooting Common Problems](#troubleshooting-common-problems)
    - [Namespace is stuck in Terminating State](#namespace-is-stuck-in-terminating-state)

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

## Configurating DNS for the Kubernetes Cluster

Setup a wildcard A entry for all three IP addresses of the k8s cluster in your DNS provider:

```
for i in {1..3}; do bash -c "ansible-inventory --list | jq '._meta.hostvars' | jq '.[\"k8s-$i\"].ipv4' | tr -d '\"'" ; done
```

For example, I have the following DNS entries:

```
*.k8s A 159.69.189.138 3600
k8s A 159.69.189.138 3600
```

## Configuring Cert Manager for the Kubernetes Cluster

Note: cert-manager v1.x.x on rke kubernetes cluster (k8s v1.17.14) does not install webhooks properly. That's the reason why we are installing v0.16.1 in this section. For the latest k8s v1.19.x, you can install the latest version of the cert-manager (v1.x.x).

Installing cert-manager:

```
export KUBECONFIG=kube_config_cluster.yml
# check that you are on the right cluster
k get no

# Install the CustomResourceDefinition resources separately
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.1

kubectl -n cert-manager rollout status deploy/cert-manager
```

Validate cert-manager installation:

```
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF

k apply -f test-resources.yaml
k delete -f test-resources.yaml
```

Install letsencrypt issuers for staging and prod:

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
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
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
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

Test letsencrypt certificates:

TODO: certificates are not getting issued from the cluster-issuer, but from the default k8s build-in CA for some reason.

```
cat <<EOF > cert-manager-test.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - test.${SERVER}
    secretName: nginx-tls
  rules:
  - host: test.${SERVER}
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
EOF

k apply -f cert-manager-test.yml
k delete -f cert-manager-test.yml
```

Install CA certificates inside the cluster:

```
export EMAIL=youremailaddress@example.com
export DOMAIN=your.domain.com
export SERVER=k8s.${DOMAIN}

mkdir tls

cat <<EOF > tls/openssl.cnf
[ req ]
#default_bits           = 2048
#default_md             = sha256
#default_keyfile        = privkey.pem
distinguished_name      = req_distinguished_name
attributes              = req_attributes

[ req_distinguished_name ]
countryName                     = DE
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = Sachsen
localityName                    = Leipzig
0.organizationName              = Ermilov Org
organizationalUnitName          = DevOps
commonName                      = ${SERVER}
commonName_max                  = 64
emailAddress                    = ${EMAIL}
emailAddress_max                = 64

[ req_attributes ]
challengePassword               = A challenge password
challengePassword_min           = 4
challengePassword_max           = 20

[ v3_ca ]
basicConstraints = critical,CA:TRUE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always
EOF

openssl genrsa -out tls/ca.key 2048
openssl req -x509 -new -nodes -key tls/ca.key -subj "/CN=${SERVER}" -days 3650 -out tls/ca.crt -extensions v3_ca -config tls/openssl.cnf

kubectl create secret tls ca-keypair --cert=tls/ca.crt --key=tls/ca.key --namespace=cert-manager
```

Install ca issuer for the cluster:

```
cat <<EOF | k apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-keypair
EOF
```


## Configuring Rancher UI for the Kubernetes Cluster

The official installation documentation is located [here](https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/).

```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
kubectl create namespace cattle-system

export EMAIL=youremailaddress@example.com
export DOMAIN=your.domain.com
export SERVER=k8s.${DOMAIN}

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=${SERVER} \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=${EMAIL}

# wait until deployment is rolled out
kubectl -n cattle-system rollout status deploy/rancher
```

## Troubleshooting Common Problems

### Namespace is stuck in Terminating State

In case you can not remove a namespace, you can try to reset its' finalizers:

```
k proxy
export NAMESPACE=cert-manager
kubectl get ns ${NAMESPACE} -o json | \
  jq '.spec.finalizers=[]' | \
  curl -X PUT http://localhost:8001/api/v1/namespaces/${NAMESPACE}/finalize -H "Content-Type: application/json" --data @-
```