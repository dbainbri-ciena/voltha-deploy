# Replicating `kind-voltha` Sanity Test

## Create `KinD` Kubernetes Cluster

### Create the Basic Cluster
**NOTE:** _This is a one time setup as there is no need to repeatedly destroy and create the Kubernetes cluster_

Using the cluster configuration file (`basic-cluster.yaml`)
```
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```
The cluster can be created with
```
kind create cluster --config <file> --name basic-cluster
export KUBECONFIG=$(kind get kubeconfig-path --name basic-cluster)
```

### Initialize Helm
```
helm init --wait
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

## Install the VOLTHA Dependencies (Infrastructure)
Using the following helm settings file will create a HA ONOS cluster and other infrastructure
bits (`dev-infra.yaml`).
```
---
onos:
  replicas: 2
  atomix:
    replicas: 3
```
The infrastructure can be installed with
```
helm install --atomic --wait --namespace infra --name dev-infra --set-file kubeconfig=$KUBECONFIG -f dev-infra.yaml deploy/voltha-environment
```

## Install a VOLTHA Stack
**NOTE:** _Currently only a single stack is supported until additional patches are merged into the VOLTHA code base_
Values file
```
---
voltha:
  services:
    controller:
      - service: dev-infra-onos-0.dev-infra-onos-hs.infra.svc.cluster.local
        port: 6653
      - service: dev-infra-onos-1.dev-infra-onos-hs.infra.svc.cluster.local
        port: 6653
      - service: dev-infra-onos-2.dev-infra-onos-hs.infra.svc.cluster.local
        port: 6653
    etcd:
      service: dev-infra-etcd.infra.svc.cluster.local
      port: 2379
    kafka:
      adapter:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092
      cluster:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092

openolt:
  services:
    etcd:
      service: dev-infra-etcd.infra.svc.cluster.local
      port: 2379
    kafka:
      adapter:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092
      cluster:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092

openonu:
  services:
    etcd:
      service: dev-infra-etcd.infra.svc.cluster.local
      port: 2379
    kafka:
      adapter:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092
      cluster:
        service: dev-infra-kafka.infra.svc.cluster.local
        port: 9092
```
Command:
```
helm install --atomic --wait --namespace stack1 --name dev-stack1 -f dev-stack.yaml deploy/voltha-stack
```

## Done
At this point everything is up and running. Port forwarding into the k8s cluster is not configured.
