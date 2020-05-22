# VOLTHA Environment and Stack Helm Charts
This repository contains Kubernetes Helm Charts that can be used to deploy
VOLTHA dependencies (environment) as well as VOLTHA stacks (Core, OF Agent,
and Adapters).

## Add Helm Chart Repository To Environment
```
helm repo add deploy https://dbainbri-ciena.github.io/voltha-deploy/charts
helm repo update
```

## Environment
### **TL;DR**
Install VOLTHA environment in the Kubernetes namespace `infra`
```
helm install --atomic --wait --namespace infra --name dev --set-file kubeconfig=$KUBECONFIG deploy/voltha-environment
```

Install VOLTHA environment infra in different namespace from bbsim simulators
```
helm install --set tags.bbsim=false --atomic --wait --namespace infra --name dev-infra --set-file kubeconfig=$KUBECONFIG deploy/voltha-environment
helm install --set tags.infra=false --atomic --wait --namespace bbsim --name dev-bbsim --set-file kubeconfig=$KUBECONFIG deploy/voltha-environment
```

Add BBSIM instance to deploy environment
```
helm upgrade --set bbsim1.enabled=true --set tags.infra=false --atomic --wait --namespace bbsim --set-file kubeconfig=$KUBECONFIG dev-bbsim deploy/voltha-environment
```

Removing
use `helm delete --purge`

### Options
Which environment components are installed can be controlled via value settings.
There are two main groups of components that can be optionally not be installed by
use of the `--set tags.<group>=true/false` command line option. Within these grouping each
component can be controlled by `--set <component>.enabled=true/false`.

#### Components
- Group: `infra`
  - kafka - _kafka message bus_
  - etcd - _etcd cluster_
  - radius - _free radius server_
  - onos - _ONOS SDN controller_
  - sadis - _BBSIM SADIS servier_
- Group: `bbsim`
  - bbsim0 - _BBSIM_
  - bbsim1 - _BBSIM_ (disabled by default)
  - bbsim2 - _BBSIM_ (disabled by default)
  - bbsim3 - _BBSIM_ (disabled by default)
  - bbsim4 - _BBSIM_ (disabled by default)

#### Cluster Sizers
##### Kafka
```
kafka.replicas: 1
kafka.zookeeper.replicaCount: 1
```
##### ETCd
```
etcd.replicas: 1
```

##### ONOS
```
onos.replicas: 1
onos.atomix.replicas: 1
```

## VOLTHA Stack
**NOTE:** _Multiple stacks cannot currently be deployed pending patches to core components
with respect to KV prefix and message topics_
### **TL;DR**
You will need to specify the endpoint information for Kafka and ETCd to the VOLTHA core
and adapters. This can be done via a custom `values.yaml` file. An example of such as
file is included in this repository as `examples/override.yaml`. This example assumes
that the infrastructure was installed in the namespace `infra` with the Helm release
name `dev-infra`.

```
helm install --name stack1 --namespace stack1 -f examples/override.yaml --atomic --wait deploy/voltha-stack
```

### Create and Enable BBSIM
Assumes a BBSIM instance (`bbsim0`) was deployed into the namespace `infra` with the
Helm release name `dev-infra`. Also assumes a port forward was established and `voltctl`
is configured to use that port forward.
```
voltctl device enable $(voltctl device create -t openolt -H dev-infra-bbsim0.infra.svc.cluster.local:50060)
```

**TL;DR**
```
kubectl port-forward -n stack1 svc/voltha-api 55555:55555
voltctl -s localhost:55555 version
```
