# Running Consul on Kubernetes

This tutorial will walk you through deploying a three (3) node [Consul](https://www.consul.io) cluster on Kubernetes.

## Overview

* Three (3) node Consul cluster using a [StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets)
* Secure communication between Consul members using [TLS and encryption keys](https://www.consul.io/docs/agent/encryption.html)

## Prerequisites

This tutorial leverages features available in Kubernetes 1.11.0 and later.

* [kubernetes](http://kubernetes.io/docs/getting-started-guides/binary_release) 1.11.x

```
gcloud container clusters create consul \
  --cluster-version 1.11.2-gke.9
```

The following clients must be installed on the machine used to follow this tutorial:

* [consul](https://www.consul.io/downloads.html) 1.4.0-rc
* [cfssl](https://pkg.cfssl.org) and [cfssljson](https://pkg.cfssl.org) 1.2

## Usage

Clone this repo:

```
git clone https://github.com/kelseyhightower/consul-on-kubernetes.git
```

Change into the `consul-on-kubernetes` directory:

```
cd consul-on-kubernetes
```

### Generate TLS Certificates

Install cfssl and cfssljson
```
wget -c -O /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
```
```
wget -c -O /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```
```
wget -c -O /usr/local/bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
```
```
chmod +x /usr/local/bin/cfssl*
```

RPC communication between each Consul member will be encrypted using TLS. Initialize a Certificate Authority (CA):

```
cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca
```

Create the Consul TLS certificate and private key:

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca/ca-config.json -profile=default ca/consul-csr.json | cfssljson -bare consul
```

At this point you should have the following files in the current working directory:

```
# ls -l |grep .pem
ca-key.pem
ca.pem
consul-key.pem
consul.pem
```

### Generate the Consul Gossip Encryption Key

[Gossip communication](https://www.consul.io/docs/internals/gossip.html) between Consul members will be encrypted using a shared encryption key. Generate and store an encrypt key:

```
GOSSIP_ENCRYPTION_KEY=$(docker run --rm -it consul:1.6.1 consul keygen)
```
```
echo ${GOSSIP_ENCRYPTION_KEY}
```

### Create the Consul Secret and Configmap

The Consul cluster will be configured using a combination of CLI flags, TLS certificates, and a configuration file, which reference Kubernetes configmaps and secrets.

Store the gossip encryption key and TLS certificates in a Secret:

```
kubectl create secret generic consul --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" --from-file=ca.pem --from-file=consul.pem --from-file=consul-key.pem
```

Store the Consul server configuration file in a ConfigMap:

```
kubectl create configmap consul --from-file=configs/server.json
```

### Create the Consul Service

Create a headless service to expose each Consul member internally to the cluster:

```
kubectl create -f services/consul.yaml
```

### Create the Consul Service Account and ClusterRole

```
kubectl create -f serviceaccounts/consul.yaml
```

```
kubectl create -f clusterroles/consul.yaml
```

### Create the Consul StatefulSet

Deploy a three (3) node Consul cluster using a StatefulSet:

```
kubectl get sc
NAME                                 PROVISIONER                       AGE
alicloud-disk-available              diskplugin.csi.alibabacloud.com   23d
alicloud-disk-efficiency (default)   diskplugin.csi.alibabacloud.com   23d
alicloud-disk-essd                   diskplugin.csi.alibabacloud.com   23d
alicloud-disk-ssd                    diskplugin.csi.alibabacloud.com   23d
```
```
vi statefulsets/consul.yaml
  ...
      containers:
        - name: consul
          # set image
          image: "consul:1.6.1"
  ...
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        # set storageClassName
        storageClassName: alicloud-disk-efficiency
```
```
kubectl create -f statefulsets/consul.yaml
```

### Create Ingress
```
yum install httpd-tools -y
```
```
htpasswd -bc consul-auth.txt admin password
```
```
kubectl create secret generic consul-secret --from-file consul-auth.txt
```
```
vi ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: consul
  annotations:
    kubernetes.io/ingress.class: "traefik"
    ingress.kubernetes.io/auth-type: "basic"
    ingress.kubernetes.io/auth-secret: "consul-secret"
spec:
  rules:
  - host: consul.kong62.com
    http:
      paths:
      - path: /
        backend:
          serviceName: consul
          servicePort: http
```
```
kubectl create -f ingress.yaml
```

Each Consul member will be created one by one. Verify each member is `Running` before moving to the next step.

```
kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
consul-0   1/1       Running   0          20s
consul-1   1/1       Running   0          20s
consul-2   1/1       Running   0          20s
```
```
kubectl get service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                            AGE
consul       ClusterIP      None            <none>          8500/TCP,8443/TCP,8400/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP   96s

```

### Verification

At this point the Consul cluster has been bootstrapped and is ready for operation. To verify things are working correctly, review the logs for one of the cluster members.

```
kubectl logs consul-0
```

The consul CLI can also be used to check the health of the cluster. In a new terminal start a port-forward to the `consul-0` pod.

```
kubectl port-forward consul-0 8500:8500
Forwarding from 127.0.0.1:8500 -> 8500
Forwarding from [::1]:8500 -> 8500
```

Run the `consul members` command to view the status of each cluster member.

```
kubectl exec -it consul-0 consul members
Node      Address            Status  Type    Build  Protocol  DC   Segment
consul-0  172.31.9.248:8301  alive   server  1.6.1  2         dc1  <all>
consul-1  172.31.10.98:8301  alive   server  1.6.1  2         dc1  <all>
consul-2  172.31.6.106:8301  alive   server  1.6.1  2         dc1  <all>
```

### Accessing the Web UI

The Consul UI does not support any form of authentication out of the box so it should not be exposed. To access the web UI, start a port-forward session to the `consul-0` Pod in a new terminal.

```
kubectl port-forward consul-0 8500:8500
```

Visit http://127.0.0.1:8500 or http://consul.kong62.com in your web browser.

![Image of Consul UI](images/consul-ui.png)

## Cleanup

Run the `cleanup` script to remove the Kubernetes resources created during this tutorial:

```
bash cleanup
```
