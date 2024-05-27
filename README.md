# amq-streams-testing

## Installation
- Openshift GitOps
- AMQ Streams

```
oc apply -f ./cluster-configuration
```
Wait till both operators are installed


Get Argo CD admin password:
```
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

## Create Argo CD application

```
oc apply -f ./argocd -n openshift-gitops
```
