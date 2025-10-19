# 🚀 Valkey Cluster Setup on Kubernetes

This guide provides a complete step-by-step process for deploying **Valkey** in **cluster mode** on Kubernetes.  
It includes the setup of namespaces, secrets, configuration, StatefulSets for replicas, and cluster initialization commands.

---

## 🧩 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Namespace and Secrets Setup](#namespace-and-secrets-setup)
4. [ConfigMap Configuration](#configmap-configuration)
5. [Replica StatefulSet Deployment](#replica-statefulset-deployment)
6. [Cluster Initialization](#cluster-initialization)
7. [Verification Steps](#verification-steps)

---

## 📘 Overview

Valkey is a high-performance in-memory key-value data store compatible with Redis.  
In this guide, you’ll learn how to deploy a **3-node master-replica Valkey cluster** using Kubernetes manifests and scripts.

---

## ⚙️ Prerequisites

Before you start, ensure you have the following:

- A running **Kubernetes cluster** (v1.23 or higher)
- `kubectl` CLI configured and connected to the cluster
- `openssl` installed for secret generation
- Sufficient resources for at least **3 master** and **3 replica pods**

---

## 🧰 Namespace and Secrets Setup

- Creates a namespace
- Generates a secure Valkey password
- Stores it as a Kubernetes secret
- Initializes the Valkey cluster and adds replicas

```bash
kubectl create namespace valkey --dry-run=client -o yaml | kubectl apply -f -

SECRET=$(openssl rand -base64 32)
echo requirepass $SECRET > /tmp/valkey-password-file.conf
echo primaryauth $SECRET >> /tmp/valkey-password-file.conf

kubectl create secret generic valkey-password \
  --from-file=valkey-password-file.conf=/tmp/valkey-password-file.conf \
  -n valkey

SECRET=$(kubectl get secret valkey-password -n valkey -o jsonpath="{.data.valkey-password-file\.conf}" | base64 -d | awk '{print $2; exit}')
echo "Valkey password: $SECRET"
```

## 🗂️ ConfigMap Configuration

Apply the Valkey-ConfigMap.yaml to configure cluster-level settings such as persistence, networking, and replication.

```bash
kubectl apply -f Valkey-ConfigMap.yaml -n valkey
```

## 🧱 Replica StatefulSet Deployment

Deploy the Master & replica StatefulSet using valkey-replicas-statefulset.yaml and valkey-replicas-statefulset.yaml file.

```bash
kubectl apply -f valkey-master-statefulset.yaml -n valkey

kubectl apply -f valkey-replicas-statefulset.yaml -n valkey
```

## 🧱 Valkey headless Services for cluster, master and replica

 Deploy the service valkey-service.yaml

 ```bash
kubectl apply -f valkey-service.yaml -n valkey
```

## 🔗 Cluster Initialization

Once all master and replica pods are running, initialize the cluster:

```bash
kubectl exec -it -n valkey valkey-masters-0 -- valkey-cli --cluster create --cluster-yes --cluster-replicas 0 \
  valkey-masters-0.valkey-masters.valkey.svc.cluster.local:6379 \
  valkey-masters-1.valkey-masters.valkey.svc.cluster.local:6379 \
  valkey-masters-2.valkey-masters.valkey.svc.cluster.local:6379 \
  --pass "$SECRET"
```

Then add replica nodes:

```bash

kubectl exec -ti -n valkey valkey-masters-0 -- \
  valkey-cli --cluster add-node \
  valkey-replicas-0.valkey-replicas.valkey.svc.cluster.local:6379 \
  valkey-masters-0.valkey-masters.valkey.svc.cluster.local:6379 \
  --cluster-slave --pass "$SECRET"

kubectl exec -ti -n valkey valkey-masters-0 -- \
  valkey-cli --cluster add-node \
  valkey-replicas-1.valkey-replicas.valkey.svc.cluster.local:6379 \
  valkey-masters-1.valkey-masters.valkey.svc.cluster.local:6379 \
  --cluster-slave --pass "$SECRET"

kubectl exec -ti -n valkey valkey-masters-0 -- \
  valkey-cli --cluster add-node \
  valkey-replicas-2.valkey-replicas.valkey.svc.cluster.local:6379 \
  valkey-masters-2.valkey-masters.valkey.svc.cluster.local:6379 \
  --cluster-slave --pass "$SECRET"

```

## Verify Cluster Status
``` bash

kubectl exec -it -n valkey valkey-masters-0 -- valkey-cli -a "$SECRET" cluster info
kubectl exec -it -n valkey valkey-masters-0 -- valkey-cli -a "$SECRET" cluster nodes

```

## Verify the roles of the pods and replication status

```bash
for x in $(seq 0 2); do echo "valkey-masters-$x"; kubectl exec -n valkey valkey-masters-$x  -- valkey-cli --pass ${SECRET} role; echo; done
for x in $(seq 0 2); do echo "valkey-replicas-$x"; kubectl exec -n valkey valkey-replicas-$x -- valkey-cli --pass ${SECRET} role; echo; done

```

## 🏁 Conclusion

You now have a fully functional Valkey Cluster running on Kubernetes!
This setup provides high availability, data replication, and scalability for production-grade workloads.
