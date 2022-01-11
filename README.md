# Ed-Fi Deployed via Kubernetes

This repository looks at deploying Ed-Fi via Google Kubernetes Engine. Historically I have maintained [this repo](https://github.com/xmarcosx/edfi-google-cloud-deployment) that contains all code necessary to deploy Ed-Fi via Google Cloud Run and Cloud SQL. Utilizing Cloud Run is the recommend route for most people looking to implement Ed-Fi. Cloud Run requires less management overhead and is a great fully managed serverless platform. If you're simply looking to deploy Ed-Fi in Google Cloud, go [there](https://github.com/xmarcosx/edfi-google-cloud-deployment).

This repo was started for a few reasons:
* I wanted to place [pgbouncer](https://www.pgbouncer.org/), a connection pooler, between the API and ODS to reduce the number of connections to the Cloud SQL instance allowing for smaller (AKA less expensive) instances.
* I wanted to deploy [Dagster](https://dagster.io/) and [Apache Superset](https://superset.apache.org/), and having all of that in one kubernetes cluster was interesting to me.
* I wanted to learn kubernetes and needed a project.

In this design the Ed-Fi ODS databases are deployed to a PostgreSQL Cloud SQL instance. Reading [this documentation from Google](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke), I felt that Cloud SQL was a better option than GKE due to the management overhead that running PostgreSQL in kubernetes would add.


![Ed-Fi](/assets/kube.png)

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

# create artifact registry repository
gcloud artifacts repositories create my-repository \
    --project=PROJECT_ID \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository";

gcloud builds submit \
    --tag us-central1-docker.pkg.dev/PROJECT_ID/my-repository/edfi-admin-app src/admin-app/.;

# create service account with access to cloud sql
gcloud iam service-accounts create cloud-sql-proxy;
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:cloud-sql-proxy@PROJECT_ID.iam.gserviceaccount.com" \
    --role=roles/cloudsql.client;

# create static external ip address
gcloud compute addresses create edfi --global;

# DEV TODO configure the dns records for your domain to point to external ip address

# create gke autopilot cluster
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

kubectl apply -f deployment-edfi-admin-app.yaml;
kubectl apply -f managed-cert-ingress.yaml;

# check status of certificate
kubectl describe managedcertificate managed-cert;

```

## Resources

* [Google Kubernetes Engine Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart#autopilot)
* Ernesto Garbarino: [Beginning Kubernetes on the Google Cloud Platform](https://www.amazon.com/Beginning-Kubernetes-Google-Cloud-Platform/dp/1484254902)
* [Ed-Fi Alliance Docker repo](https://github.com/Ed-Fi-Alliance-OSS/Ed-Fi-ODS-Docker)
* [SQL db GKE vs CLoud SQL](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke)
