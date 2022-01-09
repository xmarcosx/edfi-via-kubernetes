
```bash

# create artifact registry repository
gcloud artifacts repositories create hello-repo \
    --project=learning-kubernetes-337701 \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository";

# build  container image using Cloud Build, which is similar to running docker build and docker push, but the build happens on Google Cloud
gcloud builds submit \
    --tag us-central1-docker.pkg.dev/learning-kubernetes-337701/hello-repo/helloworld-gke .;

kubectl apply -f deployment.yaml;
kubectl apply -f service.yaml;

```
