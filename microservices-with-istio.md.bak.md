# Microservices with Istio - Hands On

## About
In this hands-on we will create a new GKE cluster with Istio enabled and deploy a microservices based app called bookshelf

After it is deployed, we will apply some load and see the pods scale automatically and down to zero when not required.

## List your GCP Projects and get your project ID
```bash
gcloud projects list
```

## Set project ID in the environment variable
```bash
export GOOGLE_CLOUD_PROJECT=FIXME
```

## Set default project
```bash
gcloud config set project $GOOGLE_CLOUD_PROJECT
```

## Save the hands-on working directory into environment variable
```bash
export HANDSON_WORKSPACE=$PWD
```

## Enable API's
```bash
gcloud services enable cloudbuild.googleapis.com sourcerepo.googleapis.com containerregistry.googleapis.com container.googleapis.com cloudtrace.googleapis.com cloudprofiler.googleapis.com logging.googleapis.com compute.googleapis.com run.googleapis.com
```

## Fetch Microservices application "Hipster Shop" code



### Move to working directory
```bash
cd $HANDSON_WORKSPACE
```

### Fetch code from Github
```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
```

### Move to application code directory
```bash
cd microservices-demo/
```

### Reset to validated version (2019/4/23)
```bash
git reset --hard f2f382f
```

## Create a new GKE cluster with Istio

```bash
export CLUSTER=gke-istio-cluster
```

```bash
export ZONE=us-east4-a
```

```bash
gcloud beta container clusters create $CLUSTER \
--zone $ZONE \
--enable-autorepair \
--username "admin" \
--machine-type "n1-standard-2" \
--image-type "COS" \
--disk-type "pd-standard" \
--disk-size "100" \
--scopes "https://www.googleapis.com/auth/cloud-platform" \
--num-nodes "4" \
--enable-cloud-logging --enable-cloud-monitoring \
--enable-ip-alias \
--network "projects/$GOOGLE_CLOUD_PROJECT/global/networks/default" \
--subnetwork "projects/$GOOGLE_CLOUD_PROJECT/regions/asia-northeast1/subnetworks/default" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio --istio-config auth=MTLS_PERMISSIVE
```

## Credentials to connect to GKE Cluster

```bash
gcloud container clusters get-credentials microservices-demo --zone $ZONE --project $GOOGLE_CLOUD_PROJECT
```

# Build and Deploy Application on Kubernetes

## Create Application Container & Deploy on GKE Cluster

```bash
cd $HANDSON_WORKSPACE/microservices-demo
```

skaffoldを利用してコンテナのビルドとレジストリへの登録を行なう。
### We will use skaffold to build the container and push it and register it in the container registry, and deploy it on Kubernetes Cluster

"gcb" profile allows building and pushing the images on Google Container Builder without requiring docker installed on the developer machine.

```bash
skaffold run -p gcb --default-repo=gcr.io/$GOOGLE_CLOUD_PROJECT
```

## Check Status

### Check Service Endpoint

```bash
kubectl get svc/frontend-external
```

### Connect with your browser

```
http://<EXTERNAL-IP>
```

```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```

## Upgrade Hipster Shop Front-end

### Modify source code of "AdService"

Edit the following file :

```
$HANDSON_WORKSPACE/microservices-demo/src/adservice/src/main/java/hipstershop/AdService.java
```

Change line #210 :

```
.put("cycling", bike) # Line 210 before change
.put("cycling", camera) # Line 210 after change
```

### Rebuild Container & Push it to Container Registry

This time we will use the traditional docker tool to build the container and register it

```bash
cd $HANDSON_WORKSPACE/microservices-demo/src/adservice/
```

Build Container and label as v2

```bash
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2 .
```

Register Container into Registry

```bash
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

In normal development pipelines we would use CI service to monitor updates on source code (on commit) and automatically build and push container to registry (skaffold would help you to implment CI/CD)

### Release the new version of Hipster Shop "AdService"

```bash
kubectl set image deployment/adservice server=gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

You could as well update the kubernetes Deployment manifest for AdService


### Check outcome


You should now see an Advertisement for Camera on the Cycling page (^_-)
```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```


# Introduce Service Mesh

## Preparation

### Check Status of Istio System

```bash
kubectl get pods --namespace=istio-system
```

### Istioによる管理の有効化

When you deploy your application using kubectl apply, the Istio sidecar injector will automatically inject Envoy containers into your application pods if they are started in namespaces labeled with istio-injection=enabled:

```bash
kubectl label namespace default istio-injection=enabled
```

We are going to stop all currently running pod so they will be restart with Istio sidecar container (Envoy proxy)

```bash
kubectl delete --all pods
```

Warning : Killing all running Pod is not recommended for real production environment as it would make your application not available. If your want to enable Istio on currently running application, Instead, you would create new Deployment with appropriate namespace and rollout them. 



## Rebuild and Deploy Hipster Shop

### Modify Source Code of AdService

Edit the following file:

```
$HANDSON_WORKSPACE/microservices-demo/src/adservice/src/main/java/hipstershop/AdService.java
```

Change line #210 :

```
.put("cycling", camera) # Line 210 before change
.put("cycling", airPlant) # Line 210 after change
```

### Rebuild Container & Push it to Container Registry


```bash
cd $HANDSON_WORKSPACE/microservices-demo/src/adservice/
```

Build Container

```bash
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3 .
```

Push to Container Registry

```bash
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3
```

### Create new Deployment file for AdService


#### Create Deployment for AdService v2

```bash
cd $HANDSON_WORKSPACE
```

Create a new file Deployment manifest "k8s-adservice-v2.yaml". Replacer "FIXME" with the name of you Google Cloud Project ID

プロジェクトIDの確認方法
```bash
echo $GOOGLE_CLOUD_PROJECT
```

use nano or vi to create and edit new file :
```bash
nano k8s-adservice-v2.yaml
```

copy the content below into the file k8s-adservice-v2.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adservice
spec:
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        version: v2
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: gcr.io/FIXME/adservice:v2
        ports:
        - containerPort: 9555
        env:
        - name: PORT
          value: "9555"
        #- name: JAEGER_SERVICE_ADDR
        #  value: "jaeger-collector:14268"
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
```

Apply the new Deployment manifest to deploy the container

```bash
kubectl apply -f k8s-adservice-v2.yaml
```

#### Create Deployment for AdService v3

```bash
cd $HANDSON_WORKSPACE
```

Create a new file Deployment manifest "k8s-adservice-v3.yaml". Replacer "FIXME" with the name of you Google Cloud Project ID

プロジェクトIDの確認方法
```bash
echo $GOOGLE_CLOUD_PROJECT
```

use nano or vi to create and edit new file :
```bash
nano k8s-adservice-v3.yaml
```

copy the content below into the file k8s-adservice-v3.yaml:


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adservice-v3
spec:
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        version: v3
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: gcr.io/FIXME/adservice:v3
        ports:
        - containerPort: 9555
        env:
        - name: PORT
          value: "9555"
        #- name: JAEGER_SERVICE_ADDR
        #  value: "jaeger-collector:14268"
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
```

Apply the new Deployment manifest to deploy the container

```bash
kubectl apply -f k8s-adservice-v3.yaml
```

## Control Traffic with Istio : create a Service Endpoint

### Let's define a DestinationRule for "AdService"

```bash
cd $HANDSON_WORKSPACE
```

Create a definition file named "istio-destinationrule-adservice.yaml"

use nano or vi to create and edit new file :
```bash
nano istio-destinationrule-adservice.yaml
```

copy the content below into the file :

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: adservice
spec:
  host: adservice
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

Deploy the Istio destination rule

```bash
kubectl apply -f istio-destinationrule-adservice.yaml
```

Confirm the destination rule has been registered

```bash
kubectl describe destinationrule/adservice
```

### Let's define VirtualService for "AdService"

```bash
cd $HANDSON_WORKSPACE
```

Create a definition file named "istio-virtualservice-adservice.yaml"

use nano or vi to create and edit new file :
```bash
nano istio-virtualservice-adservice.yaml
```

copy the content below into the file :

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: adservice
spec:
  hosts:
    - adservice
  http:
  - route:
    - destination:
        host: adservice
        subset: v2
      weight: 80
    - destination:
        host: adservice
        subset: v3
      weight: 20
```

Deploy the Istio Virtual Service

```bash
kubectl apply -f istio-virtualservice-adservice.yaml
```

Confirm the Virtual Service has been registered
```bash
kubectl describe virtualservices/adservice
```

Access the Hipster Shop with your browser and confirm that the Advertisement shown is : Camera 9/10 times and Airplant 1/10 times
```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```


# Congratulations !!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

You successfully achieved the Micrososervices Application Deployment !

For additional information on configuration parameters for Kubernetes and Istio, please refer to
[Kubernetes](https://kubernetes.io)
[Istio](https://istio.io)

## (Optional) Cleanup

以下は任意ですが、必要に応じてハンズオンで利用したリソースを削除してください。

### Delete the cluster
```bash
gcloud container clusters delete --zone=$ZONE --async --quiet $CLUSTER
```

### Unset your default project
```bash
gcloud config unset project
```


#appendix



---



