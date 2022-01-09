# K12 analytics data stack

This is my place to jot down notes as I work through learning kubernetes so I may deploy the components below.

| Component                      | Notes                                   |
| ------------------------------ | --------------------------------------- |
| Ed-Fi API, ODS, and Admin App  | The API will run in YearSpecific mode   |
| Dagster Cloud agent            |                                         |
| Apache Superset                |                                         |


## Google Kubernetes Engine

```bash

# enable kubernetes and create cluster
gcloud services enable artifactregistry.googleapis.com;
gcloud services enable container.googleapis.com;

gcloud config set compute/region us-central1;

gcloud container clusters create-auto my-cluster;

# get auth credentials so kubectl can interact with the cluster
gcloud container clusters get-credentials my-cluster;

```

## Resources

* 
