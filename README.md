# Coffee Demo
## Overview



## Setup


Create staging cluster on GKE: 
```
gcloud container clusters create gke-staging --no-enable-cloud-logging --no-enable-cloud-monitoring --num-nodes=2
```

Create service account on GKE cluster:
```
# export KUBECONFIG=
kubectl create serviceaccount spinnaker
kubectl create clusterrolebinding spinnaker-cluster-admin --clusterrole=cluster-admin --serviceaccount=default:spinnaker
GKE_TOKEN=$(kubectl get secret $(kubectl get serviceaccount spinnaker -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' )
GKE_SERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
```

Create cluster on CCP and download environment file to `~/Downloads/ccp-prod.env`. 

Install spinnaker on CCP cluster:
```
export KUBECONFIG=~/Downloads/ccp-prod.env

helm init --upgrade

kubectl create ns spinnaker

helm install --name spin-release stable/spinnaker --timeout 600 --namespace spinnaker
```


Configure Spinnaker:

```
# export KUBECONFIG=~/Downloads/ccp-prod.env

# Create a new kubeconfig file in the halyard pod:

kubectl -n spinnaker exec spin-release-spinnaker-halyard-0 -- bash -c "export KUBECONFIG=~/gke-staging.env; kubectl config set-cluster gke-staging --server $GKE_SERVER --insecure-skip-tls-verify"
kubectl -n spinnaker exec spin-release-spinnaker-halyard-0 -- bash -c "export KUBECONFIG=~/gke-staging.env; kubectl config set-credentials gke-staging --token=\"$GKE_TOKEN\""
kubectl -n spinnaker exec spin-release-spinnaker-halyard-0 -- bash -c "export KUBECONFIG=~/gke-staging.env; kubectl config set-context gke-staging --cluster gke-staging --user gke-staging"

# Add a new kubernetes account to Spinnaker's halyard configuration

kubectl -n spinnaker exec spin-release-spinnaker-halyard-0 -- bash -c "hal config provider kubernetes account add gke-staging --context gke-staging --kubeconfig-file ~/gke-staging.env  --docker-registries dockerhub --provider-version v2"

# Tell halyard to update Spinnaker's configuration:

kubectl -n spinnaker exec spin-release-spinnaker-halyard-0 -- bash -c "hal deploy apply"

# Create a NodePort for Spinnaker's UI and work out the URL:

kubectl -n spinnaker expose svc spin-deck --type NodePort  --name=spin-deck-np
SPINNAKER_URL="http://$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'):$(kubectl -n spinnaker get svc spin-deck-np -o jsonpath='{.spec.ports[0].nodePort}')"
echo "Access spinnaker UI at $SPINNAKER_URL"
```




## Running demo

