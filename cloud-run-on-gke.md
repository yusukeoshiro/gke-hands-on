# Cloud Run on GKE Hands-on

## About
In this hands-on we will create a new GKE cluster and deploy a simple sinatra app as Cloud Run on GKE app.

After it is deployed, we will apply some load and see the pods scale automatically and down to zero when not required.

## Set project ID in the environment variable
```bash
export GOOGLE_CLOUD_PROJECT=FIXME
```

## Set default project
```bash
gcloud config set project $GOOGLE_CLOUD_PROJECT
```

## Enable API's
```bash
gcloud services enable container.googleapis.com containerregistry.googleapis.com cloudbuild.googleapis.com run.googleapis.com
```



## Create a new cluster

```bash
export CLUSTER=cloud-run-cluster

export ZONE=us-east4-a

gcloud beta container clusters create $CLUSTER \
  --addons=HorizontalPodAutoscaling,HttpLoadBalancing,Istio,CloudRun \
  --machine-type=n1-standard-4 \
  --cluster-version=latest --zone=$ZONE \
  --enable-stackdriver-kubernetes --enable-ip-alias \
  --scopes cloud-platform \
  --enable-autoscaling --min-nodes "1" --max-nodes "30"
```

## Deploy a new service on the new cluster as Cloud Run on GKE
```bash
export IMAGE=gcr.io/oshiro-work-demo/sinatra-app:latest

export SERVICE_NAME=my-cloud-run-on-gke-app

gcloud beta run deploy $SERVICE_NAME \
    --project=$GOOGLE_CLOUD_PROJECT \
    --image=$IMAGE \
    --cluster=$CLUSTER \
    --cluster-location=$ZONE \
    --concurrency=2 \
    --namespace=default
```

## See if everything works!
```bash
export IP_ADDRESS="$(kubectl get service istio-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*].ip}")"

echo $IP_ADDRESS

export HOST=$SERVICE_NAME.default.example.com

curl -H "Host: $HOST" $IP_ADDRESS
```

## Checkout pods
```bash
kubectl get pods

# or alternatively to keep seeing

# watch kubectl get pods
```

## Apply large traffic and see what happens to the pods!
```bash
go get github.com/rakyll/hey

hey -host $HOST -c 50 -n 150000 \
    "http://${IP_ADDRESS?}"
```

---

## Clean up
### Clean up Cloud Run on GKE
```bash
# sorry no command!
# please head over to the console and manually delete the service
```

### Delete the cluster

```bash
gcloud container clusters delete --zone=$ZONE --async --quiet $CLUSTER
```