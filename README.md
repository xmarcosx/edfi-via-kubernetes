# K12 analytics data stack

This is my place to jot down notes as I work through learning kubernetes so I may deploy the components below.

| Component                      | Notes                                   |
| ------------------------------ | --------------------------------------- |
| Ed-Fi API and Admin App        | The API will run in YearSpecific mode. Will connect to Cloud SQL instance for ODS. pgbouncer will sit between the API and ODS for connection management.   |
| Dagster Cloud agent            | This is the worker agent that Dagster Cloud will execute jobs on.                                        |
| Apache Superset                |                                         |


## Google Kubernetes Engine

```bash

gcloud init;

# only if kubectl not installed already
gcloud components install kubectl;

gcloud services enable artifactregistry.googleapis.com;
gcloud services enable cloudbuild.googleapis.com;
gcloud services enable compute.googleapis.com;
gcloud services enable container.googleapis.com;
gcloud services enable sqladmin.googleapis.com;

gcloud config set compute/region us-central1;

# create service account with access to cloud sql
gcloud iam service-accounts create cloud-sql-proxy;
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:cloud-sql-proxy@PROJECT_ID.iam.gserviceaccount.com" \
    --role=roles/cloudsql.client;

# create static external ip address
gcloud compute addresses create edfi --global;


# create GKE autopilot cluster
gcloud container clusters create-auto my-cluster;

# get auth credentials so kubectl can interact with the cluster
gcloud container clusters get-credentials my-cluster;

# create secret storing cloud sql credentials
kubectl create secret generic cloud-sql-creds \
  --from-literal=username=postgres \
  --from-literal=password=XXXXXXXXX;

# create secret storing admin app encryption key
kubectl create secret generic edfi-admin-app-creds \
  --from-literal=key=$(/usr/bin/openssl rand -base64 32);

cd src;

# create kubernetes service account
kubectl apply -f service-account.yaml;

# bind kubernetes service account to google service account
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[default/cloud-sql-proxy]" \
  cloud-sql-proxy@PROJECT_ID.iam.gserviceaccount.com;

kubectl annotate serviceaccount \
  cloud-sql-proxy \
  iam.gke.io/gcp-service-account=cloud-sql-proxy@PROJECT_ID.iam.gserviceaccount.com;

# deploy pgbouncer with cloud sql proxy sidecar
kubectl apply -f deployment-pgbouncer.yaml;
kubectl apply -f service-pgbouncer.yaml;

# deploy edfi api and create managed cert for SSL
kubectl apply -f managed-cert.yaml
kubectl apply -f deployment-edfi-api.yaml;
kubectl apply -f service-edfi-api.yaml;
kubectl apply -f managed-cert-ingress.yaml;

# TODO Configure the DNS records for your domains to point to the IP address of the load balancer.
# Create A record pointing to external IP address.

# check status of certificate
kubectl describe managedcertificate managed-cert;

kubectl apply -f deployment-edfi-admin-app.yaml;

```


## Useful commands

```bash

kubectl apply -f deployment.yaml;
kubectl get pods;
kubectl get service;
kubectl logs pod-name container-name;
kubectl attach pod-name -c container-name;
kubectl delete deployment deployment-name;

kubectl create secret generic <YOUR-DB-SECRET> \
  --from-literal=username=<YOUR-DATABASE-USER> \
  --from-literal=password=<YOUR-DATABASE-PASSWORD> \
  --from-literal=database=<YOUR-DATABASE-NAME>;

# get postgres pod name and exec psql command on it
POD=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $1}'`
kubectl exec -it $POD -- psql -U postgres
\l
\c EdFi_Ods_2022
\dt *.*
\q

# create artifact registry repository
# gcloud artifacts repositories create my-repository \
#     --project=PROJECT_ID \
#     --repository-format=docker \
#     --location=us-central1 \
#     --description="Docker repository";

```


## Resources

* [Google Kubernetes Engine Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart#autopilot)
* Ernesto Garbarino: [Beginning Kubernetes on the Google Cloud Platform](https://www.amazon.com/Beginning-Kubernetes-Google-Cloud-Platform/dp/1484254902)
* [Ed-Fi Alliance Docker repo](https://github.com/Ed-Fi-Alliance-OSS/Ed-Fi-ODS-Docker)
* [SQL db GKE vs CLoud SQL](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke)
