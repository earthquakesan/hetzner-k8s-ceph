# Deploying Kubernetes Cluster on Baremetal with Ceph

This repository shows how to deploy k8s cluster using [rancher rke](https://rancher.com/docs/rke/latest/en/) tool on three baremetal nodes on Hetzner. The Hetzner nodes are provisioned using ansible playbooks. k8s installation is done with Rancher RKE tool. Additionally, we deploy a ceph cluster to enable automatic persistent volume provisioning in the cluster.

- [Deploying Kubernetes Cluster on Baremetal with Ceph](#deploying-kubernetes-cluster-on-baremetal-with-ceph)
  - [Necessary Tools](#necessary-tools)
  - [Cluster overview](#cluster-overview)
  - [Getting started](#getting-started)
  - [Provision the Virtual Machines](#provision-the-virtual-machines)
  - [Install Kubernetes Cluster using RKE](#install-kubernetes-cluster-using-rke)
  - [Testing Kubernetes Cluster](#testing-kubernetes-cluster)
  - [Configurating DNS for the Kubernetes Cluster](#configurating-dns-for-the-kubernetes-cluster)
  - [Configuring Cert Manager for the Kubernetes Cluster](#configuring-cert-manager-for-the-kubernetes-cluster)
  - [Configuring Rancher UI for the Kubernetes Cluster](#configuring-rancher-ui-for-the-kubernetes-cluster)
  - [Setting Up Rook/Ceph for the Kubernetes Cluster](#setting-up-rookceph-for-the-kubernetes-cluster)
  - [Troubleshooting Common Problems](#troubleshooting-common-problems)
    - [Namespace is stuck in Terminating State](#namespace-is-stuck-in-terminating-state)
    - [Checking DNS Resolution inside the Cluster](#checking-dns-resolution-inside-the-cluster)
  - [Additional Material](#additional-material)
    - [Deploying Kubernetes Cluster using Kubespray](#deploying-kubernetes-cluster-using-kubespray)

## Necessary Tools

The following tools need to be installed on your machine:

* [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for creating and provisioning the virtual machines
* [hcloud cli](https://github.com/hetznercloud/cli) for working with Hetzner Cloud - download binary from Releases and put it on your PATH
* [jq](https://stedolan.github.io/jq/download/) for parsing JSON output from e.g. ansible
* [RKE v1.2.4+](https://github.com/rancher/rke/releases) for provisioning k8s cluster
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

Note: provisioning is only necessary for RKE

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

## Install Kubernetes Cluster using RKE

Note: RKE cluster got problems with DNS resolution, trying kubespray (latest version). Also RKE installs k8s v1.17.x

Create rke configuration:

```
cat <<EOF > cluster.yml
ssh_agent_auth: true
cluster_name: cluster.local
name: cluster.local
enable_cluster_alerting: true
enable_cluster_monitoring: true
ignore_docker_version: true

kubernetes_version: v1.19.6-rancher1-1

nodes:
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
    user: root
    labels:
      worker: yes
    role: [controlplane, worker, etcd]
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-2"].ipv4' | tr -d '"')
    user: root
    labels:
      worker: yes
    role: [controlplane, worker, etcd]
  - address: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-3"].ipv4' | tr -d '"')
    user: root
    labels:
      worker: yes
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

## Testing Kubernetes Cluster

Validate kubernetes cluster:

```
# For RKE
export KUBECONFIG=kube_config_cluster.yml
# For Kubespray
export KUBECONFIG=kubespray_inventory/artifacts/admin.conf

alias k=kubectl

k get no

# Check that the pods can be scheduled

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

# Check external connectivity

kubectl run -i -t busybox --image=radial/busyboxplus:curl --restart=Never
curl google.com
exit
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

Installing cert-manager:

```
export CERT_MANAGER_VERSION=v1.1.0
# check that you are on the right cluster
k get no

# Install the CustomResourceDefinition resources separately
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.crds.yaml

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
  --version ${CERT_MANAGER_VERSION}

kubectl -n cert-manager rollout status deploy/cert-manager
```

Validate cert-manager installation (you can find resources for a specific version of cert-manager in [the docs](https://cert-manager.io/docs/installation/kubernetes/)):

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
export EMAIL=youremail

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
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
apiVersion: cert-manager.io/v1
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

Test issuing of letsencrypt certificates:

```
export SERVER=k8s.ermilov.org
cat <<EOF | k apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: test.${SERVER}-cert
spec:
  secretName: tls-cert
  duration: 24h
  renewBefore: 12h
  commonName: test.${SERVER}
  dnsNames:
  - test.${SERVER}
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
EOF

k get certificate -w
```

Test letsencrypt certificates with ingress resource:

```
export SERVER=k8s.ermilov.org
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
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
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
curl test.${SERVER} -L

k delete -f cert-manager-test.yml
```

Install CA certificates inside the cluster (optional):

```
export EMAIL=youremailaddress@example.com
export SERVER=k8s.ermilov.org

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

Note: in the interface you will see Controller Manager and Scheduler are not healthy, but they are. See [this issue with Rancher on the latest kubernetes](https://github.com/rancher/rancher/issues/29427).

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

After deployment is done successfully, navigate to your cluster and set the admin password.

TODO: check out how to set admin password automatically. 

## Setting Up Rook/Ceph for the Kubernetes Cluster

```
# Create Cluster
export ROOK_VERSION=release-1.5
export ROOK_GITHUB_RAW_PATH=https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/cluster/examples/kubernetes/ceph
k apply -f ${ROOK_GITHUB_RAW_PATH}/crds.yaml -f ${ROOK_GITHUB_RAW_PATH}/common.yaml -f ${ROOK_GITHUB_RAW_PATH}/operator.yaml
k apply -f ${ROOK_GITHUB_RAW_PATH}/cluster.yaml

# It takes about 5-8 mins for ceph cluster to be up and running

# Create Rook toolbox
k apply -f ${ROOK_GITHUB_RAW_PATH}/toolbox.yaml
k -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
ceph status

# Expose Ceph dashboard to the outside

export DASHBOARD_HOSTNAME=rook-ceph.k8s.ermilov.org
cat <<EOF | k apply -f - 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rook-ceph-mgr-dashboard
  namespace: rook-ceph
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/server-snippet: |
      proxy_ssl_verify off;
spec:
  tls:
   - hosts:
     - ${DASHBOARD_HOSTNAME}
     secretName: ${DASHBOARD_HOSTNAME}-tls
  rules:
  - host: ${DASHBOARD_HOSTNAME}
    http:
      paths:
      - path: /
        backend:
          serviceName: rook-ceph-mgr-dashboard
          servicePort: https-dashboard
EOF

# Get admin password for the dashboard
# Login at ${DASHBOARD_HOSTNAME}
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

# Provision Ceph Filesystem

export FILESYSTEM_NAME=myfs
cat <<EOF | k apply -f - 
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: ${FILESYSTEM_NAME}
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
EOF

# Wait until provisioning is finished (pods should be up and running)
kubectl -n rook-ceph get pod -l app=rook-ceph-mds

# Or check with ceph status (mds should be up)
k -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status

# Provision StorageClass

export FILESYSTEM_NAME=myfs
cat <<EOF | k apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where operator is deployed.
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: ${FILESYSTEM_NAME}

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-data0

  # Root path of an existing CephFS volume
  # Required for provisionVolume: "false"
  # rootPath: /absolute/path

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete
EOF

# Creating a PeristentVolumeClaim and write some data into it

cat <<EOF | k apply -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 1Gi
---
kind: Pod
apiVersion: v1
metadata:
  name: create-file-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
    - "/bin/sh"
    args:
    - "-c"
    - "echo Hello world! > /mnt/test.txt && cat /mnt/test.txt && exit 0 || exit 1"
    volumeMounts:
    - name: pvc
      mountPath: "/mnt"
  restartPolicy: "Never"

  volumes:
  - name: pvc
    persistentVolumeClaim:
      claimName: claim1
EOF
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

### Checking DNS Resolution inside the Cluster

```
kubectl run -i -t busybox --image=radial/busyboxplus:curl --restart=Never
```

## Additional Material

### Deploying Kubernetes Cluster using Kubespray

```
git submodule add https://github.com/kubernetes-sigs/kubespray.git kubespray
cd kubespray
# your pip and python should be version 3+
pip install -r requirements.txt
cp -r inventory/mycluster ../kubespray_inventory
cd ..

cat <<EOF > kubespray_inventory/hosts.yaml
all:
  hosts:
    node1:
      ansible_user: root
      ansible_host: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
      ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
      access_ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-1"].ipv4' | tr -d '"')
    node2:
      ansible_user: root
      ansible_host: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-2"].ipv4' | tr -d '"')
      ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-2"].ipv4' | tr -d '"')
      access_ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-2"].ipv4' | tr -d '"')
    node3:
      ansible_user: root
      ansible_host: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-3"].ipv4' | tr -d '"')
      ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-3"].ipv4' | tr -d '"')
      access_ip: $(ansible-inventory --list | jq '._meta.hostvars' | jq '.["k8s-3"].ipv4' | tr -d '"')
  children:
    kube-master:
      hosts:
        node1:
        node2:
    kube-node:
      hosts:
        node1:
        node2:
        node3:
    etcd:
      hosts:
        node1:
        node2:
        node3:
    k8s-cluster:
      children:
        kube-master:
        kube-node:
    calico-rr:
      hosts: {}
EOF

# In kubespray_inventory/group_vars/k8s-cluster/k8s-cluster.yml 
cluster_name: cluster.local

cd kubespray 
ansible-playbook -i ../kubespray_inventory/hosts.yaml cluster.yml
```