# GKE with serviceaccount

## Create Service Account

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

## Run Workload

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