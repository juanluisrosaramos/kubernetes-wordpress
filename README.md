# Kubernetes-wordpress

Migrate the installation of a website consisting in WP + Mysql to Kubernetes containers in GCE
This project is inspired in the following [google guide](https://cloud.google.com/container-engine/docs/tutorials/persistent-disk). As the objective of the project is experiment with GCE containers and **see if we can reduce costs** compairing to virtual machines, the new environment will hold a minimum number of containers (3 nodes) starting with the minimum hw resources necessary (memory and cpu) and predemptible VM machines. We also will need two persistent disk for the data (minimum of 10Gb) of the WP and the Database f the 10Gb.

## Configuring kubernetes containerized platform
Set defaults for the gcloud command-line tool

*gcloud config set project PROJECT_ID*
*gcloud config set compute/zone europe-west1-c*

Create the cluster with the minimum possible nodes
gcloud container clusters create ateiavlc --num-nodes=3 --disk-size=10  --tags=wp,mysql --machine-type=g1-small 

Create the persistent storage


## Migrating from production 
Using the plugin UpdraftPlus europe-west3that will create a copy in a Google Drive using the Google API and [activating the Drive API](https://updraftplus.com/support/configuring-google-drive-api-access-in-updraftplus/) 
