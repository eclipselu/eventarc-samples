# Eventarc AuditLog-Cloud Storage and Workflows Integration

In this sample, you will see how to connect
[Eventarc](https://cloud.google.com/eventarc/docs) events to
[Workflows](https://cloud.google.com/workflows/docs).

More specifically, you will see how a file creation in a storage bucket triggers
a Cloud Run service via Eventarc. In turn, the Cloud Run service executes a
workflow with the bucket and file name.

## Deploy a workflow

First, deploy the [workflow.yaml](../eventarc-workflows-integration/eventarc-auditlog-storage/workflow.yaml). It simply
logs out the bucket and file name for the storage event.

Deploy workflow:

```sh
export WORKFLOW_NAME=workflow-auditlog-storage
export REGION=us-central1
gcloud workflows deploy ${WORKFLOW_NAME} --source=workflow.yaml --location=${REGION}
```

## Deploy a Cloud Run service to execute the workflow

Next, deploy a Cloud Run service to execute workflow. It simply executes the
workflow with the bucket and file name. You can see the source code in
[trigger-workflow](../eventarc-workflows-integration/eventarc-auditlog-storage/trigger-workflow).

Build the container:

```sh
export PROJECT_ID=$(gcloud config get-value project)
export SERVICE_NAME=trigger-workflow-auditlog-storage
gcloud builds submit --tag gcr.io/${PROJECT_ID}/${SERVICE_NAME} .
```

Deploy the service:

```sh
gcloud config set run/region ${REGION}
gcloud config set run/platform managed
gcloud run deploy ${SERVICE_NAME} \
  --image gcr.io/${PROJECT_ID}/${SERVICE_NAME} \
  --region=${REGION} \
  --allow-unauthenticated \
  --update-env-vars GOOGLE_CLOUD_PROJECT=${PROJECT_ID},WORKFLOW_REGION=${REGION},WORKFLOW_NAME=${WORKFLOW_NAME}
```

## Connect Cloud Storage events to the Cloud Run service

Connect Cloud Storage events to the Cloud Run service by creating an Eventarc
AuditLog trigger for Cloud Storage.

First, one-time Eventarc setup:

```sh
export PROJECT_NUMBER="$(gcloud projects describe $(gcloud config get-value project) --format='value(projectNumber)')"

gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
    --member=serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
    --role='roles/eventarc.eventReceiver'
```

Create the trigger:

```sh
gcloud eventarc triggers create ${SERVICE_NAME} \
  --destination-run-service=${SERVICE_NAME} \
  --destination-run-region=${REGION} \
  --location=${REGION} \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=storage.googleapis.com" \
  --event-filters="methodName=storage.objects.create" \
  --service-account=${PROJECT_NUMBER}-compute@developer.gserviceaccount.com
```

## Trigger the workflow

Upload a file to a bucket to trigger the workflow.

Create a bucket:

```sh
export BUCKET="$(gcloud config get-value core/project)-eventarc-workflows"
gsutil mb -l $(gcloud config get-value run/region) gs://${BUCKET}
```

Create a file in the bucket:

```sh
echo "Hello World" > random.txt
gsutil cp random.txt gs://${BUCKET}/random.txt
```

In the logs, you should see that Workflow received the Cloud Storage event.
