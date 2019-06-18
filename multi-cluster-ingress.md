# Multi Cluster Ingress Hands-on

## About
In this hands-on we will create three clusters in three separate regions and deploy 3 identitcal applications.

We will then use multi-cluster ingress to have a single load balancer serve global traffic from the nearest cluster.

#### Source:

https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress

https://github.com/GoogleCloudPlatform/k8s-multicluster-ingress


## Setup Your Environement

### Find out the ID of your project
```bash
gcloud projects list
```

### Set project ID in the environment variable
```bash
export GOOGLE_CLOUD_PROJECT=FIXME
```

### Set default project
```bash
gcloud config set project $GOOGLE_CLOUD_PROJECT
```

### Set the hands on workspace path
```bash
export HANDSON_WORKSPACE=$PWD
```

### Enable API's
```bash
gcloud services enable cloudbuild.googleapis.com sourcerepo.googleapis.com containerregistry.googleapis.com container.googleapis.com cloudtrace.googleapis.com cloudprofiler.googleapis.com logging.googleapis.com compute.googleapis.com run.googleapis.com
```

## Setup Clusters

```bash
export CLUSTER1=mci-cluster-1;
export CLUSTER2=mci-cluster-2;
export CLUSTER3=mci-cluster-3;
```
```bash
gcloud container clusters create --zone=asia-northeast1-c --async $CLUSTER1;
gcloud container clusters create --zone=us-east4-a --async $CLUSTER2;
gcloud container clusters create --zone=europe-west1-c --async $CLUSTER3;
```

#### Check clusters are running
```bash
gcloud container clusters list
```

use `watch` command to keep watching
```bash
watch gcloud container clusters list
```

Continue to the next step AFTER all clusters are provisioned!

## Create kubeconfig file containing credentials for all the clusters
```bash
cd $HANDSON_WORKSPACE
```

Output the crednetial for mci-cluster-1 ~ 3
```bash
KUBECONFIG=$HANDSON_WORKSPACE/mcikubeconfig gcloud container clusters get-credentials --zone=asia-northeast1-c $CLUSTER1;

KUBECONFIG=$HANDSON_WORKSPACE/mcikubeconfig gcloud container clusters get-credentials --zone=us-east4-a $CLUSTER2;

KUBECONFIG=$HANDSON_WORKSPACE/mcikubeconfig gcloud container clusters get-credentials --zone=europe-west1-c $CLUSTER3;
```

Confirm that mcikubeconfig contains the credential for 3 clusters
```bash
cat $HANDSON_WORKSPACE/mcikubeconfig
```

## Download the kubemci command-line tool and make sure it is executable
```bash
cd $HANDSON_WORKSPACE;
wget https://storage.googleapis.com/kubemci-release/release/latest/bin/linux/amd64/kubemci
chmod +x ./kubemci
```

## Clone hands-on repository
```bash
cd $HANDSON_WORKSPACE;
git clone https://github.com/GoogleCloudPlatform/k8s-multicluster-ingress.git;
cd k8s-multicluster-ingress;
git reset --hard ef552c2;
cd examples/zone-printer;
ls -l
```

## Deploy sample app on all three clusters
```bash
for ctx in $(kubectl config get-contexts -o=name --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig); do kubectl --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig --context="${ctx}" create -f manifests/ ; done
```

check pods are running fine
```bash
for ctx in $(kubectl config get-contexts -o=name --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig); do
  kubectl --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig --context="${ctx}" get pods ;
done
```

## Create Multi Cluster Ingress
### Reserve IP Address
```bash
ZP_KUBEMCI_IP=zp-kubemci-ip;
gcloud compute addresses create --global $ZP_KUBEMCI_IP";
```

Check it is reserved
```bash
gcloud compute addresses list
```

### Fix ingress.yaml to use the reserved static ip address
```bash
sed -i -e "s/\$ZP_KUBEMCI_IP/${ZP_KUBEMCI_IP}/" ingress/ingress.yaml;
```

Check the change made
```bash
git diff ingress/ingress.yaml
```


### Deploy multicluster ingress
```bash
$HANDSON_WORKSPACE/kubemci create zone-printer \
    --ingress=ingress/ingress.yaml \
    --gcp-project=$GOOGLE_CLOUD_PROJECT \
    --kubeconfig=$HANDSON_WORKSPACE/mcikubeconfig
```

### Check status
```bash
$HANDSON_WORKSPACE/kubemci get-status zone-printer --gcp-project=$GOOGLE_CLOUD_PROJECT
```
Confirm that a page is served properly at http://IP_ADDRESS

Note: It may take a several minutes for the global load balancer to get ready!

---

## Clean up
#### delete load balancer
```bash
$HANDSON_WORKSPACE/kubemci delete zone-printer \
    --ingress=ingress/ingress.yaml \
    --gcp-project=$GOOGLE_CLOUD_PROJECT \
    --kubeconfig=$HANDSON_WORKSPACE/mcikubeconfig
```
#### delete deployments and services
```bash
for ctx in $(kubectl config get-contexts -o=name --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig); do kubectl --kubeconfig $HANDSON_WORKSPACE/mcikubeconfig --context="${ctx}" delete -f manifests/ ; done
```


#### delete ip address and clusters
```bash
gcloud compute addresses delete --global --quiet "${ZP_KUBEMCI_IP}";
gcloud container clusters delete --zone=asia-northeast1-c --async --quiet $CLUSTER1;
gcloud container clusters delete --zone=us-east4-a --async --quiet $CLUSTER2;
gcloud container clusters delete --zone=europe-west1-c --async --quiet $CLUSTER3;
```
