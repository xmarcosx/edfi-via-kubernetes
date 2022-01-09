# K12 analytics data stack

This is my place to jot down notes as I work through learning kubernetes so I may deploy the components below.

| Component                      | Notes                                   |
| ------------------------------ | --------------------------------------- |
| Ed-Fi API, ODS, and Admin App  | The API will run in YearSpecific mode   |
| Dagster Cloud agent            |                                         |
| Apache Superset                |                                         |


## Google Kubernetes Engine

```bash

gcloud init;
gcloud components install kubectl;

gcloud services enable artifactregistry.googleapis.com;
gcloud services enable cloudbuild.googleapis.com;
gcloud services enable compute.googleapis.com;
gcloud services enable container.googleapis.com;

gcloud config set compute/region us-central1;

gcloud container clusters create-auto my-cluster;

# get auth credentials so kubectl can interact with the cluster
gcloud container clusters get-credentials my-cluster;

# create artifact registry repository
gcloud artifacts repositories create my-repository \
    --project=learning-kubernetes-337701 \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository";

```

## Useful commands

```bash

kubectl apply -f deployment.yaml;
kubectl get pods;
kubectl get service;
kubectl logs container-name;
kubectl attach pod-name -c container-name;
kubectl delete deployment deployment-name;

# get postgres pod name and exec psql command on it
POD=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $1}'`
kubectl exec -it $POD -- psql -U postgres

```


## Resources

* [Google Kubernetes Engine Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart#autopilot)
* Ernesto Garbarino: [Beginning Kubernetes on the Google Cloud Platform](https://www.amazon.com/Beginning-Kubernetes-Google-Cloud-Platform/dp/1484254902)
* [Ed-Fi Alliance Docker repo](https://github.com/Ed-Fi-Alliance-OSS/Ed-Fi-ODS-Docker)
* [SQL db GKE vs CLoud SQL](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke)
