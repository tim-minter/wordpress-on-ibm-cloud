# Installing Wordpress on IBM Cloud with NGINX, SSL and the ability to scale (low cost method)

First thing's first. This is a solution to get Wordpress up and running with SSL and the ability to scale. That is pretty much what you need for a medium size production ready system. So I'd call this a "minimum viable production" version. This is better than you get "out of the box", in two huge ways - SSL and scalability. Do please note that an enterprise production version would be more complex and involve high availability architecture via globally disbursed infrastructure and a load balancer solution, all of which increase the cost dramatically. 

I think this is great solution that allows you to start with one Wordpress node and scale horizontally by adding more nodes when you need/can afford.

This is how this solution compares: 

| Feature | Basic | This solution (multi worker) | This solution (single worker) | Enterprise Production Requirements |
| --- | --- | --- | --- | --- |
| Link to instructions | [Link](https://cloud.ibm.com/catalog/content/wordpress::1-Qml0bmFtaS13b3JkcHJlc3M=-global) | See below | See below | [Link](https://cloud.ibm.com/catalog/content/wordpresspro-Qml0bmFtaS13b3JkcHJlc3Nwcm8=-global) |
| Secure connection from browser (SSL) | No | Yes | Yes | Yes |
| Database | Local low performance | Local with caching (1) | Local with caching (1) | Managed High performance Database |
| Scalable | No | Yes | Yes | Yes|
| Caching (required to handle any decent load) | No | Yes | Yes | Yes |
| Proper High availability | No (this is min spec and will likely fail if a worker is removed) | No (but hardware fault tollerant if more workers are added or higher spec workers are used)| No | Yes |
| Min CPU requirements | 4 | 6 | 4 |Depends |
| Min memory requirements | 8Gb | 12Gb | 32Gb | Depends |
| Min number of worker nodes and spec(2) | 2 (2 CPU 4GB RAM Each) | 3 (2 CPU 4GB RAM Each) | 1 (4 CPU 16GB RAM) | 3 (Depends) |
| Recommended number of worker nodes and spec(3) | 3 (4 CPU 16GB RAM Each) | 3 (8 CPU 32GB RAM Each) | 1 (8 CPU 32GB RAM) | 3 (Depends) |

(1) If you want to use a high performance managed database in place of the mariadb installed locally by default, this is relatively easy to do just by providing the credentials and connection details in the wordpress install step below.

(2) It is possible to install Wordpress on IBM Cloud on a one node cluster and it runs fine using the instructions below, but you can't scale it horizontally of course. It does mean you can start with one node and then add another, and so on however. So could be great for getting started with min cost and then scaling up! The min spec for this solution seems to be 4CPU and 32Gb RAM. If you do this, you'll need to scale down a number of deployments to 1. Check for failed deployments in the Kubernetes Dashboard.

(3) For wordpress, generally the number of CPUs has the greatest impact on performance so go for machines that have the highest number of CPU that fit in your budget. 

Wordpress with NGINX and SSL is available in the IBM Cloud catalogue so why create this article? That solution in IBM Cloud is what might be termed an enterprise production solution, as discussed above. If you search for Wordpress SSL in the IBM Cloud catalogue you'll be presented with a setup page that requires a Virtual Machine cluster and a separate Virtual Machine. This is needed if you are going to have a load ballanced, potentially highly available solution... that is some significant infrastructure and may be out of your price range, especially if you are just after a production ready Wordpress instance for your business. For you business site you will certainly need SSL, and scalability is key.

It should also be noted that the "standard" Wordpress installation in IBM Cloud doesn't come with any SSL capability. These instructions come from my experience trying to get the standard Wordpress installation production ready, by adding SSL and getting it to accept incoming connections for www.example.com and example.com and making it scalable for future expansion or to handle load peaks.

What I ended up doing, I believe, is creating the equivalent of the Wordpress with NGINX and SSL service from the standard Wordpress service but without the complication of any high availability infrastructure. These instructions also benefit from a lot effort trying to find the right process to enable scaling via the use of the right storage solution.

This service is provisioned on top of Kubernetes. That could put you off, however there are significant advantanges to running Wordpress on Kubernetes, as opposed to running on it on your own server:
1. There are no Windows or Linux servers to look after, secure, patch or fix if something crashes. 
2. If a new version of Kubernetes comes along you can perform an update with a couple of clicks from the Kubernetes Dashboard. This can happen without interuption to your service if you have spare capacity on your workers (or add a worker for this purpose during the update process).
3. Your system can be horizontally scaled (easily) or verically scaled (fairly easily) and this can be automatic. 
4. Upgrading Wordpress itself can be done via a simple command.

## Steps

1. Log in to cloud.ibm.com or create your account
2. You'll need a pay as you go account to provision servers so you'll need to upgrade by entering your card details. When you come to create your "cluster" you'll be asked to upgrade if you haven't already done so.
3. Ignore the Wordpress offerings in the catalogue (these provide either the basic installation or the enterprise installation - neither of which we are interested in right now. See links in the table above) and instead create a Kubernetes cluster on Classic infrastructure.
4. In the cloud dashboard at cloud.ibm.com search for "Kubernetes" and select "Kubernetes Service".
5. Ensure that classic Infrastructure is selected, change Geography to your desired deployment area and select Single Zone in the Availability drop down, then choose your worker zone (data center). As you can see, this is where you could choose to deploy your cluster across the world or across a country or continent however we will be deploying to one data center today.
6. In the worker pool section you can select the number of workers and the size (cost) of those workers. See the table above for the min requirements. Ideally you would be creating a 3 node cluster with workers having the highest CPU count you can afford. Note that there are 3 different types of server you can select - shared, dedicated and bare metal. With dedicated and bare metal you have access to all the resources all the time. With shared servers you should/could have access to all the resources all the time but this is not guaranteed. Shared servers rely on the "neighbours" you are sharing the underlaying hardware with, not running at 100% all the time. Shared servers are the cheapest and suitable for workloads that vary, ie not workloads that run at 80-100% all the time.
7. Give you cluster a name in the last field, check the estimated cost on the right hand side and check/adjust your workers to achieve the correct cost then click Create. The cluster creation will take some time.
8. If not already installed, install the Block Storage for VPC plugin via the Getting Started page for your cluster.
9. Check/view your cluster by clicking on the Kubernetes Dashboard button from the Overview page of your cluster. Keep this dashboard open for use later. 
10. Connect from your command line/terminal to your cluster as described in the Access page of your cluster. You need to install helm [Link](https://helm.sh/docs/intro/install/) and the IBM Cloud Command line tool (as described in the Access page), if you haven't already done this (one time process). 

The following steps 11 to 17 are taken from, or modified from the Bitnami documentation here https://docs.bitnami.com/tutorials/secure-wordpress-kubernetes-managed-database-ssl-upgrades/ 

11. Install the NGINX ingress controller
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install ingress bitnami/nginx-ingress-controller
```
12. Obtain the LoadBalancer IP address
```
kubectl get svc ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```
13. Configure the DNS for your domain name by adding an A record pointing to the public IP address obtained above. This may take a few minutes or a few hours to take effect

14. Add the cert-manager repository, create a namespace and create CRDs
```
helm repo add jetstack https://charts.jetstack.io
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml
```
15. Create a ClusterIssuer resource for Let's Encrypt certificates. Create a file named letsencrypt-prod.yaml with the following content. Replace the EMAIL-ADDRESS placeholder with a valid email address.
```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  labels:
    name: letsencrypt-prod
spec:
  acme:
    email: EMAIL-ADDRESS
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: nginx
```
16. Apply the changes to the cluster
```
kubectl apply -f letsencrypt-prod.yaml
```
17. Install cert-manager with Helm and configure Let's Encrypt as the default Certificate Authority (CA)
```
helm install cert-manager --namespace cert-manager jetstack/cert-manager --version v0.14.1
```
18. Now we get the requirements for scalability. Storage that supports ReadWriteMany (RWX) access is a requirement for this. Block Storage (we added the block storage plugin earlier) doesn't support this natively. It took a lot of trial and error to find a simple solution to this and essentially one (there may be more) solution is to install a NFS file server on your cluster which then provides NFS access (which supports RWX) to one of the IBM Kubernetes block storage classes. In the case below I have chosen the "ibmc-vpc-block-retain-general-purpose" storage class. This is the fastest available storage (there are also ibmc-block-bronze and ibmc-block-silver classes that provide access to slightly cheaper storage).
19. Install a NFS server via the following process:

Create a file called values.yaml in your local machine. Paste the following into the file and save.
```
persistence:
  enabled: true
  storageClass: "ibmc-vpc-block-retain-general-purpose"
  size: 20Gi

storageClass:
  defaultClass: true
```
Issue the following commands to create the nfs server on your cluster

```
helm repo add kvaps https://kvaps.github.io/charts
helm install kvaps/nfs-server-provisioner -f values.yaml --generate-name

```
19. Install WordPress using Bitnami's Helm chart with additional parameters to integrate with Ingress and cert-manager. Replace the YOURDOMAIN placeholder with your domain name. There are critical differences between this command and the command in the Bitnami docs. The differences set the persistent storage class (for Wordpress files) to use the NFS server created above, set the Wordpress database to use the ibmc-block-gold storage class and enable caching properly which is absolutely critical to the overall performance of Wordpress.

```
helm install --set service.type=ClusterIP --set ingress.enabled=true --set ingress.certManager=true --set ingress.annotations."kubernetes\.io/ingress\.class"=nginx --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod --set ingress.hostname=YOURDOMAIN --set ingress.extraTls[0].hosts[0]=YOURDOMAIN --set ingress.extraTls[0].secretName=wordpress.local-tls --set persistence.accessMode=ReadWriteMany --set persistence.storageClass=nfs --set wordpressConfigureCache=true --set memcached.enabled=true --set allowOverrideNone=true --set htaccessPersistenceEnabled=true --set mariadb.primary.persistence.storageClass=ibmc-vpc-block-retain-general-purpose wordpress bitnami/wordpress
```
## Notes on some of the settings in the command above:
Setting these options turns on db caching and installs the w3 totalcache plugin using memcached which is highly recommended. See https://github.com/bitnami/charts/tree/master/bitnami/wordpress

```
--set memcached.enabled=true
--set wordpressConfigureCache=true
```

Setting this option makes the .htaccess file persistent and editable by plugins. See here for more into and if you should do this or not https://docs.bitnami.com/kubernetes/apps/wordpress/configuration/understand-htaccess/

```--set htaccessPersistenceEnabled=true``` 

Note: All possible settings are described here https://github.com/bitnami/charts/blob/master/bitnami/wordpress/values.yaml

Note: Stuffed up your installation and can't log on etc? Just issue ```helm uninstall wordpress```, delete the Persistent Volume Claim for the mariadb and then issue the command above again.... that's it! 



20. The cert-manager will go off to the Let's Encrypt service and generate SSL certificates for you and then manage them for you! If your DNS update hasn't taken effect yet, cert-manager can't go and make that request and you'll see an extra pod running in your Kubernetes dashboard until it can go and do this.
21. When this all completes you should be able to access Wordpress via your domain name. It may take a while for this complete.
22. The W3 Total Cache plugin will be installed and generally set up. Review the settings and note that memcached is set/available in the list of caching servers. 
23. Now what about if you want to be able to connect your Wordpress instance using www.mydomain.com as well as mydomain.com. This is a common requirement. Searching for this on the internet will return a lot of scary looking processes but in this case, basically nginx and cert-manager take care of this, you just need to add a few lines to the ingress definition....

In your Kubernetes Dashboard in IBM Cloud go to the Ingresses menu under Service. Edit the Wordpress ingress. Under the Spec section you will see something like this...

```
spec:
  tls:
    - hosts:
        - mydomain.co.uk
      secretName: wordpress.local-tls
  rules:
    - host: mydomain.co.uk
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              serviceName: wordpress
              servicePort: http
```

to add the "www" version of your domain change it to this kind for format...

```
spec:
  tls:
    - hosts:
        - www.mydomain.co.uk
        - mydomain.co.uk
      secretName: wordpress.local-tls
  rules:
    - host: www.mydomain.co.uk
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              serviceName: wordpress
              servicePort: http
    - host: mydomain.co.uk
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              serviceName: wordpress
              servicePort: http
```

Save and close.

## Scaling this solution
So how do you scale this? Two options; manually and automatically. You'll probably want to use both options.

### Manually
You can test this out if you have a one node cluster but for it to be useful you either need a cluster with more than one node (worker) already, or you need to add one or more workers to your single worker cluster. These can be added by "resizing" you worker pool via the IBM Cloud dashboard. Note that you can only add nodes of the same size as the ones you already have. If you want to add bigger or smaller nodes, you can do this but you need to add a new worker pool. The workers in a pool all have to be the same essentially. Generally it is best to have fewer larger workers than it is to have lots of smaller ones.

Once you have additional nodes you can scale your Wordpress instance by issuing the following command
```
kubectl scale --replicas=3 deployment/wordpress
```
You can watch this change in your Kubernetes dashboard via the Deployments page. You can also manually change the number of pods (scale) the deployment there via the three dots menu.

### Automatically
You may want to set the min number of pods using the method above and then, in production you will want scaling to be handled for you based on load. This is where a Kubernetes Horizontal Autoscaler comes in.
Info about this can be found here for now https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

On top of this, you can also have IBM Cloud automatically create/destroy workers for you so that there are workers to scale onto. To add this functionality go to the Add-ons section of your Kubernetes service in IBM Cloud.

## Some important Notes
### File Upload Limit Errors
By default the file upload limit of the ingress probably wont be sufficient which will mean you get errors when you upload anything over that limit (even though the upload limit you can see in the Wordpress dashboard will be 40Mb or similar) To change the file upload size limit go to your Kubernetes dashboard and then go to Ingresses under the Service section and locate the Wordpress ingress. Edit the ingress and add the following two lines to the annotations section.

```
nginx.ingress.kubernetes.io/proxy-body-size: 100m
nginx.org/client-max-body-size: 100m
```
eg.
```apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: <RESOURCE_NAME>
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    kubernetes.io/tls-acme: 'true'
    cert-manager.io/cluster-issuer: letsencrypt-prod
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.org/client-max-body-size: 100m
spec:
  tls:
  - hosts:
    - <HTTPS_HOSTNAME>
    secretName:  <RESOURCE_NAME>
  rules:
  - host: <HTTPS_HOSTNAME>
    http:
      paths:
      - backend:
          serviceName: <BAKCEND_SERVICE_NAME>
          servicePort: <BACKEND_SERVICE_PORT>
        path: /
   ```     
### Reset admin/user password

Connect to a Wordpress pod either by launching the "exec" console on the wordpress pod within the Kubernetes dashboard or by using the kubectl command line. 
Once "in" the pod you basically see something like what you'd see if Wordpress was running on a single server. Change directory to the /bitnami/wordpress folder. The command we are going to run just updates the database.

Once connected to the console of the pod (via the Kubernetes dashboard UI or shh) and issuing a ls command you see the following...

```I have no name!@wordpress-766c568f5:/$ ls
apache-init.sh      bin      build  home   media                     opt           proc  sbin  tmp  wordpress-init.sh
apache-inputs.json  bitnami  dev    lib    mnt                       post-init.d   root  srv   usr  wordpress-inputs.json
app-entrypoint.sh   boot     etc    lib64  mysql-client-inputs.json  post-init.sh  run   sys   var
```
All the expected Wordpress stuff is inside the bitmani folder

```I have no name!@wordpress-766c568f5:/$ cd bitnami/
I have no name!@wordpress-766c568f5:/bitnami$ ls
apache  gosu  libphp  php  tini  wordpress  wp-cli
I have no name!@wordpress-766c568f5:/bitnami$ cd wordpress/
I have no name!@wordpress-766c568f5:/bitnami/wordpress$ ls
wp-config.php  wp-content
```
Here we can use the ```wp``` command like we would on a server

```I have no name!@wordpress-766c568f5:/bitnami/wordpress$ wp user list
+----+------------+--------------+------------------+---------------------+---------------+
| ID | user_login | display_name | user_email       | user_registered     | roles         |
+----+------------+--------------+------------------+---------------------+---------------+
| 1  | user       | user         | user@example.com | 2021-04-01 15:48:41 | administrator |
+----+------------+--------------+------------------+---------------------+---------------+
```
To update the password of the admin user, locate its ID (1 in the case above) and issue the command below using your own password of course

```I have no name!@wordpress-766c568f5:/bitnami/wordpress$ wp user update 1 --user_pass=newpasswordhere        
sh: 1: /usr/sbin/sendmail: not found
Success: Updated user 1.
I have no name!@wordpress-766c568f5:/bitnami/wordpress$
```
The line
```sh: 1: /usr/sbin/sendmail: not found```
is expeced because, initially, mailing is not configured.

### Modifying Wordpress files eg wp-admin.php

I've not had to do this in practice once I'd figured out the the RWX persistent storage issues and got my cluster running as above, so this info is included here just as an FYI... 

Connect to a Wordpress pod using the method above or the kubctl command line.
Once "in" the pod you basically see something like what you'd see if Wordpress was running on a single server. The /bitnami/wordpress folder is mounted to persistent storage so we can modify the contents of this folder. 

Once connected to the console of the pod (via the Kubernetes dashboard UI or shh) and issuing a ls command you see the following...

```I have no name!@wordpress-766c568f5:/$ ls
apache-init.sh      bin      build  home   media                     opt           proc  sbin  tmp  wordpress-init.sh
apache-inputs.json  bitnami  dev    lib    mnt                       post-init.d   root  srv   usr  wordpress-inputs.json
app-entrypoint.sh   boot     etc    lib64  mysql-client-inputs.json  post-init.sh  run   sys   var
```
All the expected Wordpress stuff is inside the bitmani folder. Here you can update permissions (to allow plugins to edit them if you get errors about that) on the files and view them (via the cat command). If you want to edit the files that is a different matter because there is no editor such as nano or vi available in the pod. You'll need to Google that.
