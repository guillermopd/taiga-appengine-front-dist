# Taiga Google App Engine Frontend Deployment

This is a forked version of the great Taiga project management tool developed by Kaleidos, kudos to them!
https://github.com/taigaio/taiga-front-dist

This repo contains small changes to allow the deployment of Taiga in Google App Engine

## Why

Google App Engine allows us to deploy Taiga in an elastic serverless way so we don't worry about scaling the application.
As an added benefit, when the application is not in use it will be scaled down to 0 without instance cost.

## Deployment

### Pre-requisites

- Working python3.7 install with pip and venv
- Google Cloud SDK and gcloud utility installed and authenticated: https://cloud.google.com/sdk/gcloud

### System preparation

<details><summary>Install psycopg system dependencies</summary>
<p>

```shell script
sudo apt install libpq-dev
```

</p>
</details>

<details><summary>Install gcloud alpha components (used for tying the project to a billing account)</summary>
<p>

```shell script
gcloud components install alpha
```

</p>
</details> 


### Google Project creating and configuration

Most of these steps can be executed by point-and click on the gcp web interface, however at th end of the section we provide
an sscript to automate these steps:

1. Create a new project
2. Enable Billing on the project
3. Enable Google App Engine and choose a region to serve from
4. Create a postgreSQL Cloud SQL instance
5. Create a taiga user on the SQL instance
6. Create a taiga database on the SQL instance
7. Create a Google Storage bucket to save file uploads from the users
8. Create a Service Account with access to the bucket and save the JSON authentication file

##### First we define some environemnt variables we'll be referencing 

```shell script
export DJANGO_SETTINGS_MODULE='settings.gae'

# these vars are not passed to gae
export GOOGLE_CLOUD_PROJECT='taiga-gae-tests'
export GAE_SERVICE='taiga-backend'

export REGION='europe-west'
export DB_REGION='europe-west1'

export SQL_NAME='taiga-gae-db'
export SQL_TIER='db-f1-micro'
export SQL_USER='taiga'
export SQL_PASS='taiga'

export GS_BACKEND_BUCKET='taiga-backend-files'
export GS_FRONTEND_BUCKET='taiga-frontend-files'

export SERVICE_ACCOUNT_NAME='taiga-gs-sa'

export GCP_BILLING_ACCOUNT=''
```

Adjust these values to suit your needs. Billing account id can be found with

```shell script
$ gcloud alpha billing accounts list
```



```shell script
gcloud projects create $GOOGLE_CLOUD_PROJECT
gcloud alpha billing projects link $GOOGLE_CLOUD_PROJECT --billing-account $GCP_BILLING_ACCOUNT
gcloud app create --region $REGION --project $GOOGLE_CLOUD_PROJECT

gcloud sql instances create $SQL_NAME --tier $SQL_TIER --region $DB_REGION --database-version POSTGRES_11 --storage-type SSD --project $GOOGLE_CLOUD_PROJECT
gcloud sql users create $SQL_USER --instance $SQL_NAME --password $SQL_PASS --project $GOOGLE_CLOUD_PROJECT
gcloud sql databases create taiga --instance $SQL_NAME --project $GOOGLE_CLOUD_PROJECT

gsutil mb -b on -c STANDARD -l $DB_REGION -p $GOOGLE_CLOUD_PROJECT gs://$GS_BACKEND_BUCKET

gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --project $GOOGLE_CLOUD_PROJECT
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member "serviceAccount:$SERVICE_ACCOUNT_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" --role "roles/storage.admin" --project $GOOGLE_CLOUD_PROJECT
gcloud iam service-accounts keys create bucket-access.json --iam-account $SERVICE_ACCOUNT_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --project $GOOGLE_CLOUD_PROJECT
```

### Deploy the default service

Clone this repository

```shell script
git clone https://github.com/guillermopd/taiga-appengine-front-dist.git
```

Before deploying we have to obtain the backend URL of the service, we can obtain this by deploying taiga-appengine-back
or following this instructions:
https://cloud.google.com/appengine/docs/standard/nodejs/communicating-between-services

Once we know the backend endpoint URL we add it to the file dist/conf.json, leaving /api/v1/ at the end.

```shell script
"api": "https://<<BACKEND_URL>>/api/v1/",
```

Now we're ready to deploy the frontend:

```shell script
gcloud app deploy --project $GOOGLE_CLOUD_PROJECT -v v-1589301622195
```