# Environment Setup

Obtain `project_id` of your project:
```
gcloud config get-value project
```

View details of your project:
```
gcloud compute project-info describe --project $(gcloud config get-value project)
```

Setup default region and zone:
```
gcloud config set compute/region <region>
gcloud config set compute/zone <zone>
```

Setup common variables:
```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=$(gcloud config get-value compute/region)
export ZONE=$(gcloud config get-value compute/zone)
```

Alernative way:
```
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export PROJECT_NUMBER=$(gcloud projects list --filter=projectId:$PROJECT_ID \
  --format="value(projectNumber)")
```

Enable Google APIs (For Vertax AI):
```
gcloud services enable \
  compute.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  notebooks.googleapis.com \
  aiplatform.googleapis.com \
  bigquery.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com
```

# IAM

Obtain an authorization token using your account:
```
gcloud auth print-access-token
```

# Calling Google REST API

Call API with Bearer Authorization Token
```
curl -s \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  <api_url> \
  <api_params>
```

# Cloud Storage

Create a new bucket
```
REGION="us-east4"
BUCKET="gs://gcp-cloud-storage-bucket"
gsutil mb -l $REGION $BUCKET
```

Download files to current directory:
```
gcloud storage cp gs://path/to/file .
```
or
```
gsutil cp -r gs://path/to/file .
```

Upload files to Cloud Storage:
```
gsutil cp local_file gs://bucket_name
```

# Cloud Storage Functions

Enable neccessary APIs:
```
gcloud services enable \
  artifactregistry.googleapis.com \
  cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com \
  eventarc.googleapis.com \
  run.googleapis.com \
  logging.googleapis.com \
  pubsub.googleapis.com
```


Create a Cloud Storage function:
1. SETUP - Grant the pubsub.publisher IAM role to the Cloud Storage service account.
```
PROJECT_NUMBER=$(gcloud projects list --filter="project_id:$PROJECT_ID" --format='value(project_number)')
SERVICE_ACCOUNT=$(gsutil kms serviceaccount -p $PROJECT_NUMBER)

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$SERVICE_ACCOUNT \
  --role roles/pubsub.publisher
```

2. CREATE - Prepare the cloud function app
```
mkdir ~/hello-storage && cd $_
touch index.js && touch package.json
```
Add corresponding contents in your index.js and package.json files, e.g.
```index.js
const functions = require('@google-cloud/functions-framework');

functions.cloudEvent('helloStorage', (cloudevent) => {
  console.log('Cloud Storage event with Node.js in GCF 2nd gen!');
  console.log(cloudevent);
});
```
```package.json
{
  "name": "nodejs-functions-gen2-codelab",
  "version": "0.0.1",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/functions-framework": "^2.0.0"
  }
}
```

3. DEPLOY - Deploy the function
```
gcloud functions deploy nodejs-storage-function \
  --gen2 \
  --runtime nodejs18 \
  --entry-point helloStorage \
  --source . \
  --region $REGION \
  --trigger-bucket $BUCKET \
  --trigger-location $REGION \
  --max-instances 1
```

4. TEST
Test the function by uploading a file to the bucket:
```
echo "Hello World" > random.txt
gsutil cp random.txt $BUCKET/random.txt
```
Run the following command. You should see the received CloudEvent in the logs:
```
gcloud functions logs read nodejs-storage-function \
  --region $REGION --gen2 --limit=100 --format "value(log)"
```

Delete the cloud function:
```
gcloud run services delete nodejs-storage-function --region $REGION
```

# Compute Engine

Create a E2 standard virtual machine named `myvm`:
```
gcloud compute instances create myvm --zone $ZONE --machine-type e2-standard-2
```

Create a new persistant disk named `mydisk`:
```
gcloud compute disks create mydisk --size=200GB \
--zone $ZONE
```

Attach disk `mydisk` to Virtual Machine `myvm`:
```
gcloud compute instances attach-disk myvm --disk mydisk --zone $ZONE
```

Connect your VM with ssh:
```
gcloud compute ssh myvm --zone $ZONE
```

# Network

Create new network:
```
gcloud compute networks create labnet --subnet-mode=custom
```

Create subnet:
```
gcloud compute networks subnets create labnet-sub \
   --network labnet \
   --region $REGION \
   --range 10.0.0.0/28
```

Create firewall rule:
```
gcloud compute firewall-rules create labnet-allow-internal \
	--network=labnet \
	--action=ALLOW \
	--rules=icmp,tcp:22 \
	--source-ranges=0.0.0.0/0
```


# GKE

Create a GKE cluster:
```
gcloud container clusters create --machine-type=e2-medium --zone=$ZONE lab-cluster
```

Authenticate with the GKE cluster:
```
gcloud container clusters get-credentials lab-cluster
```

Create a new Deployment `hello-server` from the sample container image:
```
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
```

Expose the service:
```
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```

# Cloud SQL

Connect to your instance:
```
gcloud sql connect myinstance --user=root
```


# BigQuery

Show help:
```
bq help <command>
```

Show the schema of a table:
```
bq show <project:public dataset.table>
```

Run a query, e.g:
```
bq query --use_legacy_sql=false \
'SELECT
   word,
   SUM(word_count) AS count
 FROM
   `bigquery-public-data`.samples.shakespeare
 WHERE
   word LIKE "%raisin%"
 GROUP BY
   word'
```

List all datasets in current project:
```
bq ls
```

Create a new dataset:
```
bq mk <dataset>
```

Create and load a table:
```
bq load dataset.table data.csv name:string,gender:string,count:integer
```

Remove a dataset:
```
bq rm -r dataset
```

# AI

Bulk update cloud storage path of data csv file.
```
sed -i -e "s/original_path/${BUCKET}/g" ./data.csv
```
