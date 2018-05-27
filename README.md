# GKE with serviceaccount

## Create Node's Service Account

```shell
export NODE_SA_NAME=gke-node-sa

gcloud iam service-accounts create $NODE_SA_NAME --display-name "Node Service Account"
export NODE_SA_EMAIL=`gcloud iam service-accounts list --format='value(email)' --filter='displayName:Node Service Account'`

export PROJECT=`gcloud config get-value project`

gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/monitoring.metricWriter
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/monitoring.viewer
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/logging.logWriter

gcloud config set container/new_scopes_behavior true
```

## Create Cluster

```shell
gcloud beta container clusters create gke-serviceaccount-test \
  --service-account=$NODE_SA_EMAIL \
  --zone=$ZONE \
  --cluster-version=$VERSION
```

## Create CI/CD's Service Account

```shell
# Create service account
export CICD_SA_NAME=gke-cicd-sa
gcloud iam service-accounts create $CICD_SA_NAME --display-name "CI/CD Service Account"
export CICD_SA_EMAIL=`gcloud iam service-accounts list --format='value(email)' --filter='displayName:CI/CD Service Account'`

# Bind service account policy
export PROJECT=`gcloud config get-value project`

gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:${CICD_SA_EMAIL} --role=roles/container.developer

# Create service account key and activate it
gcloud iam service-accounts keys create \
    /home/$USER/key.json \
    --iam-account $CICD_SA_EMAIL

gcloud auth activate-service-account $CICD_SA_EMAIL --key-file=key.json
```

`get-credentials` appends to the `kubeconfig` file, the context for the GKE master we want to auth against, it's formatted like this `gke_$PROJECT_$ZONE_$CLUSTER_NAME`

```shell
GOOGLE_APPLICATION_CREDENTIALS="/home/$USER/key.json" gcloud container clusters get-credentials gke-serviceaccount-test --zone $ZONE --project $PROJECT
```

### Token Alternative

had a use case where I needed to use tokens. instead of generated credentials, skip to [Testing Context](#testing-context)

```shell
# Generate token from key
export GOOGLE_APPLICATION_CREDENTIALS="/home/$USER/key.json"
gcloud beta auth application-default print-access-token > /home/$USER/token
```

add the token to your `/home/$USER/.kube/config`, remove auth-provider from your config file

```diff
18,26c18
<                 "auth-provider": {
<                     "name": "gcp",
<                     "config": {
<                         "cmd-args": "config config-helper --format=json",
<                         "cmd-path": "/google/google-cloud-sdk/bin/gcloud",
<                         "expiry-key": "{.credential.token_expiry}",
<                         "token-key": "{.credential.access_token}"
<                     }
<                 }
---
>                 "token": "<TOKEN>"
```

Then add the content of your token, `$context_user` in this case, is the user that was generated by `get-context`

```shell
kubectl config set-credentials $context_user --token=$(cat /home/$USER/token)
```

### Testing Context

context was generated using `gcloud container clusters`

```shell
GOOGLE_APPLICATION_CREDENTIALS="/home/$USER/key.json" gcloud container clusters get-credentials gke-serviceaccount-test --zone $ZONE --project $PROJECT
```

there's no need to activate the context since `gcloud container clusters get-credentials` did it for you, but just to be explicit, list context avaialable to you first

```shell
kubectl config get-contexts
```

activate the context you want this way, where `$context_name` is the context generated by get credentials, at the time of this tutorial, it's formatted like this `gke_$PROJECT_$ZONE_$CLUSTER_NAME` using variables already set.

```shell
kubectl config use-context $context_name
```

use the token to delete a random item

```shell
kubectl get pods
kubectl delete pods/$podname --namespace $namespace
```

verify in stackdriver

```shell
resource.type="k8s_cluster"
resource.labels.location="$ZONE"
resource.labels.cluster_name="gke-serviceaccount-test"
protoPayload.methodName:"delete"
protoPayload.resourceName="core/v1/namespaces/$namespace/pods/$podname"
```

## Run Workload

this part of the tutorial is assuming workload is pulling an image from gcr inside the same project `gcr.io/$PROJECT/$PREFIX/web-app`

```shell
gcloud container clusters get-credentials gke-serviceaccount-test --zone $ZONE --project $PROJECT

kubectl run web-app --image=gcr.io/$PROJECT/$PREFIX/web-app 
```

### Workload Fails (ImagePullBackOff)

```shell
POD_NAME=`kubectl get pods -o jsonpath='{.items[?(@.metadata.labels.run=="web-app")].metadata.name}'`

$ kubectl describe pod $POD_NAME
...
Events:
  Type     Reason                 Age               From                                                          Message
  ----     ------                 ----              ----                                                          -------
  Normal   Scheduled              7m                default-scheduler                                             Successfully assigned web-app-764784b488-kcgvv to gke-gke-serviceac
count-t-default-pool-a262a520-7dw5
  Normal   SuccessfulMountVolume  7m                kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  MountVolume.SetUp succeeded for volume "default-token-t8sg8"
  Normal   Pulling                5m (x4 over 7m)   kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  pulling image "gcr.io/$PROJECT/$PREFIX/web-app"
  Warning  Failed                 5m (x4 over 7m)   kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  Failed to pull image "gcr.io/$PROJECT/$PREFIX/web-app": rpc er
ror: code = Unknown desc = Error response from daemon: repository gcr.io/$PROJECT/$PREFIX/web-app not found: does not exist or no pull access
  Warning  Failed                 5m (x4 over 7m)   kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  Error: ErrImagePull
  Normal   BackOff                5m (x6 over 7m)   kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  Back-off pulling image "gcr.io/$PROJECT/$PREFIX/web-app"
  Warning  Failed                 2m (x20 over 7m)  kubelet, gke-gke-serviceaccount-t-default-pool-a262a520-7dw5  Error: ImagePullBackOff
```

### Setting right pull permissions

NOTE: `CICD_SA` was not configured to be able to set iam permissions, these steps are done via the same admin account that created `CICD_SA`

<!--
here's how I wish it worked
```shell
BUCKET_PATH=artifacts.$PROJECT.appspot.com/containers/repositories/library/$PREFIX/
gsutil acl ch -r -u $NODE_SA_EMAIL:R gs://$BUCKET_PATH
```-->

<!--
This is not recommended
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/storage.objectViewer
-->

```shell
BUCKET_NAME=artifacts.$PROJECT.appspot.com/
gsutil iam ch serviceAccount:$NODE_SA_EMAIL:objectViewer gs://$BUCKET_NAME

POD_NAME=`kubectl get pods -o jsonpath='{.items[?(@.metadata.labels.run=="web-app")].metadata.name}'`

$ kubectl delete pod $POD_NAME
```