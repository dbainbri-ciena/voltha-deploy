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
kind create cluster --config examples/sanity/basic-cluster.yaml --name basic-cluster
export KUBECONFIG=$(kind get kubeconfig-path --name basic-cluster)
```

### Initialize Helm
**NOTE:** _`https://dbainbri-ciena.github.io/charts` is used as the `onf` repo because it has
versions of the ONF charts that are not yet committed._
```
helm init --wait
helm repo add onosproject http://charts.onosproject.org
helm repo add ciena-incubator https://dbainbri-ciena.github.io/charts/
helm repo add onf https://dbainbri-ciena.github.io/charts/
helm repo add kubernetes-incubator https://kubernetes-charts-incubator.storage.googleapis.com
helm repo update
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
helm install --atomic --wait --namespace infra --name dev-infra --set-file sadis.kubeconfig=$KUBECONFIG -f examples/sanity/dev-infra.yaml ciena-incubator/voltha-environment
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
helm install --atomic --wait --namespace stack1 --name stack1 -f examples/sanity/dev-stack.yaml ciena-incubator/voltha-stack
```

## Done
At this point everything is up and running. Port forwarding into the k8s cluster is not configured.

## Extra Credit: Create, Enable, Authenticate, and Assign IP
```
kubectl port-forward -n stack1 svc/stack1-voltha-api 55555:55555 >/dev/null 2>&1 &
kubectl port-forward -n infra  svc/dev-infra-onos-hs 8101:8101 >/dev/null 2>&1 &
voltctl device enable $(voltctl device create -t openolt -H dev-infra-bbsim0.infra.svc:50060)
```

Repeat the following command until the device is in the `AUTHORIZED_STATE`
```
ssh -p 8101 karaf@localhost aaa-users
```

```
ssh -p 8101 karaf@localhost volt-add-subscriber-access of:00000a0a0a0a0a00 16
```

Repeat the following command until DHCPACK is received
```
ssh -p 8101 karaf@localhost dhcpl2relay-allocations
```

## Cleanup
```
kill %1 %2
helm delete --purge dev-infra stack1
kind delete cluster --name basic-cluster
```
