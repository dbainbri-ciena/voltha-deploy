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
