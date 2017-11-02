# Kubernetes-wordpress

Migrate the installation of a website consisting in WP + Mysql to Kubernetes containers in GCE
This project is copied from the following [google guide](https://cloud.google.com/container-engine/docs/tutorials/persistent-disk). As the objective of the project is experiment with GCE containers and **see if we can reduce costs** compairing to virtual machines, the new environment will hold a minimum number of containers (3 nodes) starting with the minimum hw resources necessary (memory and cpu) and predemptible VM machines. We also will need two persistent disk for the data (minimum of 10Gb) of the WP and the Database f the 10Gb.

## Results of the experiment
Finnally, after testing the kubernetes installation in production the costs nearly doubled the VM architecture. We decided to create the minimum host architecture necessary for the project and the predemptible hosts worked well and cheap but the costs increase due to the load-balancer (needed for having an external IP) provided by GCE and with an excessive cost as it is a very very simple website that doesn't need load balancing etc.. For a simple Wordpress website Kubernetes is not recommended.

## Configuring kubernetes containerized platform
Set defaults for the gcloud command-line tool

*gcloud config set project PROJECT_ID*
*gcloud config set compute/zone europe-west1-c*

Create the cluster with the minimum possible nodes
gcloud container clusters create ateiavlc --num-nodes=3 --disk-size=10  --tags=wp,mysql --machine-type=g1-small 

If you create the cluster with the GCE console (as you cannot create Pre-emptible nodes with the command tool) you can connect to it. Run the following command to retrieve cluster credentials and configure kubectl command-line tool with them:

$ gcloud container clusters get-credentials CLUSTERNAME

Create the persistent storage
*gcloud compute disks create --size 10GB mysql-disk*
*gcloud compute disks create --size 10GB wordpress-disk*

At this moment we have 50gb in use divided in 5 disks of 10Gb. One for each node and two for persisten storage. 10Gb is the minimum acceptable for GCE and they warn us of a poor I/O performance. The VM instalation just have a 20GB disk for all the instalation.

### Set up MySQL
#### Deploy the mysql container
Firstly create a Kubernetes Secret to store the password for the database. *kubectl create secret generic mysql --from-literal=password=YOUR_PASSWORD* Some characters as | are not accepted probably [a limitation of the CLI](https://kubernetes.io/docs/concepts/configuration/secret/). In the console we can found this key in configuration-->secret details. We cannot edit it 

This create secret is called from the manifest that creates the mysql instance. 

            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
                  
The deployment manifest YAWL file is in [this](https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/mysql.yaml) github project and it deployes the last available version at the moment 5.7 the path for the container is mountPath: /var/lib/mysql but we connect the persistent disk volume created before in the section of volumes.

To deploy this manifest file, run: *kubectl create -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/mysql.yaml*. We checked it was succesfully created with kubectl get pod -l app=mysql

#### Create the Mysql service
Now we have to create a service that deploys a Cluster IP ClusterIP: Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType and make it visible in the mysql port number 3306
*kubectl create -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/mysql-service.yaml*
We can get details of the service like the ip calling *kubectl get service mysql*

### Set up Wordpress
#### Deploy the WP container
Now we going to deploy the last worpress in a single instance pod reading the environment variable WORDPRESS_DB_PASSWORD and accessing to the MYSQL host in 3306 and referring to it by his name "mysql", because of Kubernetes DNS allows Pods to communicate a Service by its name.

kubectl create -f  https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/wp.yaml

Let's check the pod with *kubectl get pod*
NAME                         READY     STATUS    RESTARTS   AGE
mysql-3368603707-rdq92       1/1       Running   0          1h
wordpress-3479901767-ztpsr   1/1       Running   0          1h

#### Opening to the world
This step has been the most exhausting and the only difficulty that we find in this project. We tried different options as none seems to work and finnally a load-balancer was used and worked smoothly but it increased the costs of the project to a non acceptable budget (the loadbalance cost was the same as the rest of the project (3 predemptible hosts and on static IP).

**Non working solution**
*kubectl expose deployment wordpress --target-port=80  --type=NodePort*
Container Engine makes your Service available on a randomly-selected high port number (e.g. 32640) on all the nodes in your cluster. To make your HTTP(S) web server application publicly accessible, you need to create an Ingress resource.
generate a static (and global) IP  *gcloud compute addresses create NAME-ip --global* and assign this static IP to an **Ingress service**
kubectl create -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/basic-ingress.yaml


Firstly create a **TLS certificate**
*$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/O=nginxsvc"*
Generating a 2048 bit RSA private key writing new private key to 'tls.key'
Second upload it to kubernetes
*$ kubectl create secret tls tls-secret --key tls.key --cert tls.crt*
secret "tls-secret" created

Secondly generate a **Test HTTP Service pod** that generates a NodePort and a Replication Controller We can deploy it as follows *kubectl create -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/http-svc.yaml*
gcloud compute addresses list --global

Thirdly generate a static (and global) IP  *gcloud compute addresses create NAME-ip --global* and assign this static IP to an **Ingress service**

We need to create a load balancer (subject* to billing) servicewith the command *kubectl create -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/wp-service.yaml* we will use a static ip when we move to production. Make note of the External IP using kubectl get services. 
At this point we going to Cloud Platform Console -> VPCNetwork -> External IP Addresses and reserve as static the external ip that was reservedd in the previous step and modify it int the wp-service.yaml maniphesto adding *loadBalancerIP:IPADDRESS* in the LoadBalancer specification. Then we redeploy the service with *kubectl apply -f https://raw.githubusercontent.com/juanluisrosaramos/kubernetes-wordpress/master/wp-service.yaml*

## Migrating from production 
Using the plugin UpdraftPlus europe-west3that will create a copy in a Google Drive using the Google API and [activating the Drive API](https://updraftplus.com/support/configuring-google-drive-api-access-in-updraftplus/) and then restoring this files in the new wp.

##Measuring results.
Using [the tool Pingdom](https://tools.pingdom.com) we found this results
Performance grade B 82 / Load time 1.46 s / Faster than 81 % of tested sites / Page size  1.0 MB / Requests 71 / Tested from  Stockholm on Aug 29 at 17:29
[Capacity test](https://app.loadimpact.com/load-test/6095abf1-1d5b-4b5b-8e9d-4e870ab83b2d?charts=type%3D1%3Bsid%3D__li_bandwidth%3A1%3BdataKey%3Davg%3B%3Btype%3D1%3Bsid%3D__li_requests_per_second%3A1%3BdataKey%3Davg%3B%3Btype%3D1%3Bsid%3D__li_connections_active%3A1%3BdataKey%3Dvalue%3B%3Btype%3D8%3Bsid%3D__li_loadgen_cpu_utilization%3A1%3BdataKey%3Dvalue%3B%3Btype%3D8%3Bsid%3D__li_loadgen_memory_utilization%3A1%3BdataKey%3Dvalue%3B%3Btype%3D1%3Bsid%3D__li_failure_rate%3A1%3BdataKey%3Davg%3B%3Btype%3D1%3Bsid%3D__li_user_load_time%3A1%3BdataKey%3Dvalue&large-charts=type%3D1%3Bsid%3D__li_clients_active%3A1%3BdataKey%3Dvalue%3B%3Btype%3D1%3Bsid%3D__li_user_load_time%3A1%3BdataKey%3Dvalue)
[This tool GTMetrix](https://gtmetrix.com/reports/www.ateiavlc.org/bsDr9Flb) From Vancouver it shows Performance Scores PageSpeed Score (72%) YSlow Score (69%) Page Details Fully Loaded Time
8.6s Total Page Size 1.01MB Requests 72 
