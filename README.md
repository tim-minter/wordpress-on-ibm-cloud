# Installing Wordpress on IBM Cloud with NGINX, SSL and the ability to scale (low cost method)

First thing's first. This is a solution to get Wordpress up and running with SSL and the ability to scale. That is pretty much what you need for a production ready system but I'm not suggesting this is an enterprise grade production version. I'd call this a minimum viable production version. Better than you get "out of the box" in two huge ways - SSL and scalability. An enterprise production version would be more complex and involve high availability architecture via globally disbursed infrastructure and a load balancer solution. 

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
| Number of worker nodes(2) | 2 (2 CPU 4GB RAM Each) | 3 (2 CPU 4GB RAM Each) | 1 (4 CPU 32GB RAM) | 3 (Depends) |

(1) If you want to use a high performance managed database in place of the mariadb installed locally by default, this is relatively easy to do just by providing the credentials and connection details in the wordpress install step below.

(2) It is possible to install Wordpress on IBM Cloud on a one node cluster and it runs fine using the instructions below, but you can't scale it horizontally of course. It does mean you can start with one node and then add another, and so on. Great for getting started with min cost and then scaling up! The min spec for this solution seems to be 4CPU and 32Gb RAM. If you do this, you'll need to scale down a number of deployments to 1. Check for failed deployments in the Kubernetes Dashboard.

Wordpress with NGINX and SSL is available in the IBM Cloud catalogue so why create this article? That solution in IBM Cloud is what might be termed an enterprise production solution, as discussed above. If you search for Wordpress SSL in the IBM Cloud catalogue you'll be presented with a setup page that requires a Virtual Machine cluster and a separate Virtual Machine. This is needed if you are going to have a load ballanced, potentially highly avaiable solution... that is some significant infrastructure and may be out of your price range, especially if you are just after a production ready Wordpress instance for your business. For you business site you will certainly need SSL, and scalability is key.

It should also be noted that the "standard" Wordpress installation in IBM Cloud doesn't come with any SSL capability. These instructions come from my experience trying to get the standard Wordpress installation production ready, by adding SSL and getting it to accept incoming connections for www.example.com and example.com and making it scalable for future expansion or to handle load peaks.

What I ended up doing, I believe, is creating the equivalent of the Wordpress with NGINX and SSL service from the standard Wordpress service but without the complication of any high availability infrastructure. These instructions also benefit from a lot effort trying to find the right process to enable scaling via the use of the right storage solution.

This service is provisioned on top of Kubernetes. That could put you off, however there are significant advantanges to running Wordpress on Kubernetes, as opposed to running on it on your own server:
1. There are no Windows or Linux servers to look after, secure, patch or fix if something crashes. 
2. If a new version of Kubernetes comes along you can perofmr an update with a couple of clicks from the Kubernetes Dashboard. This can happen without interuption to your service if you have spare capacity on your workers (or add a worker for this purpose during the update process).
3. Your system can be horizontally scaled (easily) or verically scaled (fairly easily) and this can be automatic. 
4. Upgrading Wordpress itself can be done via a simple command.

## Steps

1. Log in to cloud.ibm.com or create your account
2. You'll need a pay as you go account to provision servers so you'll need to upgrade by entering your card details. When you come to create your "cluster" you'll be asked to upgrade there if you haven't already done so.
3. Ignore the Wordpress offerings in the catalogue and instead create a Kubernetes cluster on Classic infrastructure.
4. Add the IBM Block Storage plugin to your cluster (type block in the search field and select IBM Block Storage Plugin, select your cluster and click Create)
5. Connect from your command line/terminal to your cluster as described in the Access page of the cluster you created
6. Install helm and the IBM Cloud Command line tool, if you haven't already done this (one time process)

The following steps 8 to 13 are taken from, or modified from the Bitnami documentation here https://docs.bitnami.com/tutorials/secure-wordpress-kubernetes-managed-database-ssl-upgrades/ 

8. Install the NGINX ingress controller
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install ingress bitnami/nginx-ingress-controller
```
7. Obtain the LoadBalancer IP address
```
kubectl get svc ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```
8. Configure the DNS for your domain name by adding an A record pointing to the public IP address obtained above. This may take a few minutes or a few hours to take effect

9. Add the cert-manager repository, create a namespace and create CRDs
```
helm repo add jetstack https://charts.jetstack.io
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml
```
10. Create a ClusterIssuer resource for Let's Encrypt certificates. Create a file named letsencrypt-prod.yaml with the following content. Replace the EMAIL-ADDRESS placeholder with a valid email address.
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
11. Apply the changes to the cluster
```
kubectl apply -f letsencrypt-prod.yaml
```
12. Install cert-manager with Helm and configure Let's Encrypt as the default Certificate Authority (CA)
```
helm install cert-manager --namespace cert-manager jetstack/cert-manager --version v0.14.1
```
13. Install NFS server

Create a file called values.yaml. Paste the following into the file and save.
```
persistence:
  enabled: true
  storageClass: "ibmc-block-gold"
  size: 20Gi

storageClass:
  defaultClass: true
```
Issue the following commands to create the nfs server on your cluster

```
helm repo add wso2 https://helm.wso2.com
helm install stable/nfs-server-provisioner -f values.yaml --generate-name

```
14. Install WordPress using Bitnami's Helm chart with additional parameters to integrate with Ingress and cert-manager. Replace the YOURDOMAIN placeholder with your domain name. The critical difference between this and the Bitnami docs is it has settings to use the nfs storage class created above. A ReadWriteMany storage option (like NFS) is required to enable scaling.
```
helm install --set service.type=ClusterIP --set ingress.enabled=true --set ingress.certManager=true --set ingress.annotations."kubernetes\.io/ingress\.class"=nginx --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod --set ingress.hostname=YOURDOMAIN --set ingress.extraTls[0].hosts[0]=YOURDOMAIN --set ingress.extraTls[0].secretName=wordpress.local-tls --set persistence.accessMode=ReadWriteMany --set persistence.storageClass=nfs --set wordpressConfigureCache=true --set memcached.enabled=true --set allowOverrideNone=true --set htaccessPersistenceEnabled=true --set mariadb.primary.persistence.storageClass=ibmc-block-gold wordpress bitnami/wordpress
```
Note: Setting the below options turns on db caching and installs the w3 totalcache plugin using memcached which is highly recommended. See https://github.com/bitnami/charts/tree/master/bitnami/wordpress
```
--set memcached.enabled=true
--set wordpressConfigureCache=true
```

```--set htaccessPersistenceEnabled=true``` 
This makes the .htaccess file persistent and editable by plugins. See here for more into and if you should do this or not https://docs.bitnami.com/kubernetes/apps/wordpress/configuration/understand-htaccess/

Note: All possible settings are described here https://github.com/bitnami/charts/blob/master/bitnami/wordpress/values.yaml

Note: Stuffed up your installation and can't log on etc? Just issue ```helm uninstall wordpress```, delete the Persistent Volume Claim for the mariadb and then issue the command above again.... that's it!


15. The cert-manager will go off to the Let's Encrypt service and generate SSL certificates for you and then manage them for you! If your DNS update hasn't taken effect yet, cert-manager can't go and make that request and you'll see an extra pod running in your Kubernetes dashboard until it can go and do this.
16. When this all completes you should be able to access Wordpress via your domain name. It may take a while for this complete.
17. The W3 Total Cache plugin will be installed and generally set up. Review the settings and note that memcached is set/available in the list of caching servers. 
18. Now what about if you want to be able to connect your Wordpress instance using www.mydomain.com as well as mydomain.com. This is a common requirement. Searching for this on the internet will return a lot of scary looking processes but in this case, basically nginx and cert-manager take care of this, you just need to add a few lines to the ingress definition....

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
