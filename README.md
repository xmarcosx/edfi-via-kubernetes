# Ed-Fi Deployed via Kubernetes

This repository looks at deploying Ed-Fi via Google Kubernetes Engine. Historically I have maintained [this repo](https://github.com/xmarcosx/edfi-google-cloud-deployment) that contains all code necessary to deploy Ed-Fi via Google Cloud Run and Cloud SQL. Utilizing Cloud Run is the recommend route for most people looking to implement Ed-Fi. Cloud Run requires less management overhead and is a great fully managed serverless platform. If you're simply looking to deploy Ed-Fi in Google Cloud, go [there](https://github.com/xmarcosx/edfi-google-cloud-deployment).

This repo was started for a few reasons:
* I wanted to place [pgbouncer](https://www.pgbouncer.org/), a connection pooler, between the API and ODS to reduce the number of connections to the Cloud SQL instance allowing for smaller (AKA less expensive) instances.
* I plan to deploy [Dagster](https://dagster.io/) and [Apache Superset](https://superset.apache.org/) as well, and having all of that in one kubernetes cluster was interesting to me.
* I wanted to learn kubernetes and needed a project.

In this design the Ed-Fi ODS databases are deployed to a PostgreSQL Cloud SQL instance. Reading [this documentation from Google](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke), I felt that Cloud SQL was a better option than GKE due to the management overhead that running PostgreSQL in kubernetes would add.


![Ed-Fi](/assets/kube.png)

| Ed-Fi Component     | Version         |
|---------------------|-----------------|
| Ed-Fi API & ODS     | v5.3            |
| Ed-Fi Admin App     | v2.3.2          |

This repo is designed to be cloned to Google Cloud Shell and all commands can be run from there.
## Cloud SQL
```bash

gcloud services enable artifactregistry.googleapis.com;
gcloud services enable compute.googleapis.com;
gcloud services enable container.googleapis.com;
gcloud services enable sqladmin.googleapis.com;
gcloud services enable cloudbuild.googleapis.com;
gcloud services enable servicenetworking.googleapis.com;

# download database backup files
bash cloud_sql/download_db_backups.sh;

gcloud compute addresses create google-managed-services-default \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --description="peering range" \
    --network=default;

gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default \
    --network=default \
    --project=$GOOGLE_CLOUD_PROJECT;

# create cloud sql instance
gcloud beta sql instances create \
  --zone us-central1-c \
  --database-version POSTGRES_11 \
  --memory 7680MiB \
  --cpu 2 \
  --storage-auto-increase \
  --network=projects/$GOOGLE_CLOUD_PROJECT/global/networks/default \
  --no-assign-ip \
  --backup-start-time 08:00 edfi-ods-db;

gcloud sql databases create 'EdFi_Admin' --instance=edfi-ods-db;
gcloud sql databases create 'EdFi_Security' --instance=edfi-ods-db;
gcloud sql databases create 'EdFi_Ods_2023' --instance=edfi-ods-db;
gcloud sql databases create 'EdFi_Ods_2022' --instance=edfi-ods-db;
gcloud sql databases create 'EdFi_Ods_2021' --instance=edfi-ods-db;

gcloud sql users set-password postgres --password <POSTGRES_PASSWORD> --instance=edfi-ods-db;

# connect to cloud sql instance via cloud sql proxy
# keep proxy running while executing the next command
cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:5432;

bash cloud_sql/seed_databases.sh <POSTGRES_PASSWORD>;

```

## Google Kubernetes Engine

```bash

gcloud config set compute/region us-central1;

# create artifact registry repository
gcloud artifacts repositories create my-repository \
    --project=$GOOGLE_CLOUD_PROJECT \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository";

gcloud builds submit \
    --tag us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/edfi-admin-app kubernetes/admin-app/.;

# create service account with access to cloud sql
gcloud iam service-accounts create cloud-sql-proxy;
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member="serviceAccount:cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
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
  --from-literal=password=<POSTGRES_PASSWORD>;

# create secret storing admin app encryption key
kubectl create secret generic edfi-admin-app-creds \
  --from-literal=key=$(/usr/bin/openssl rand -base64 32);

cd kubernetes;

# create kubernetes service account
kubectl apply -f service-account.yaml;

# bind kubernetes service account to google service account
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:$GOOGLE_CLOUD_PROJECT.svc.id.goog[default/cloud-sql-proxy]" \
  cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com;

kubectl annotate serviceaccount \
  cloud-sql-proxy \
  iam.gke.io/gcp-service-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com;

# deploy pgbouncer with cloud sql proxy sidecar
# DEV TODO replace <INSTANCE_CONNECTION_NAME> in deployment-pgbouncer.yaml
kubectl apply -f deployment-pgbouncer.yaml;
kubectl apply -f service-pgbouncer.yaml;

# deploy edfi api and create managed cert for ssl
# DEV TODO replace domain names in managed-cert.yaml;
kubectl apply -f managed-cert.yaml;

kubectl apply -f deployment-edfi-api.yaml;
kubectl apply -f service-edfi-api.yaml;

# DEV TODO replace <GOOGLE_PROJECT_ID> in deployment-edfi-admin-app.yaml
# DEV TODO replace <DOMAIN_NAME> in deployment-edfi-admin-app.yaml
kubectl apply -f deployment-edfi-admin-app.yaml;
kubectl apply -f service-admin-app.yaml;
kubectl apply -f managed-cert-ingress.yaml;

# check status of certificate
# wait until certificate status says ""
kubectl describe managedcertificate managed-cert;

```

## Resources

* [Google Kubernetes Engine Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart#autopilot)
* Ernesto Garbarino: [Beginning Kubernetes on the Google Cloud Platform](https://www.amazon.com/Beginning-Kubernetes-Google-Cloud-Platform/dp/1484254902)
* [Ed-Fi Alliance Docker repo](https://github.com/Ed-Fi-Alliance-OSS/Ed-Fi-ODS-Docker)
* [SQL db GKE vs CLoud SQL](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke#understanding_options_to_deploy_a_database_instance_in_gke)
