# Installing Wordpress on IBM Cloud with NGINX and SSL (low cost method)


NOTE - As of 26/4/2021 I'm updatng this to include scaling with NFS. Please check back later.

The instcutions below are currently in flux so best not t0 use them!

-----------

First thing's first. This is an "in-between" solution to get Wordpress up and running with SSL and good performance. A large scale (and scalable) production version is going to be more complex. This is how this solution compares: 

Feature| Basic | This solution | Production Requirements |
| --- | --- | --- | --- |
| Wordpress up and running | Yes | Yes | Yes |
| Secure connection from browser (SSL) | No | Yes | Yes |
| Database | Local low performance | Local low performance (1) | Managed High performance Database |
| Scalable | No | No (1) | Yes|
| Caching (required to handle any decent load) | No | Yes | Yes |
| Proper High availability | No | No | Yes |
| Min CPU requirements | 4 | 6 | Depends |
| Min memory requirements | 8Gb | 12Gb | Depends |
| Number of worker nodes(2) | 2 (2 CPU 4GB RAM Each) | 3 (2 CPU 4GB RAM Each). | 3 (Depends) |

(1) both of these features can be added to/upgraded with this solution manually.
(2) It is possible to install Wordpress on IBM Cloud on a one node cluster. The min spec for this solution seems to be 4CPU and 32Gb RAM. If you do this, you'll need to scale down a number of deployments to 1. Check for failed deployments in the Kubernetes Dashboard 


Wordpress with NGINX and SSL is available in the IBM Cloud catalogue however from experience it may be easier to create an equivalent Wordpress instance yourself. If you search for Wordpress SSL in the cloud catalogue you'll be presented with a setup page that requires a Virtual Machine cluster and a separate Virtual Machine. That is some significant infrastructure and may be out of your price range especially if you are just after a production ready Wordpress instance for your business. For you business site you will certainly need SSL.

It should also be noted that the "standard" Wordpress installation doesn't come with any SSL capability. These instructions come from my experience trying to get the standard Wordpress installation production ready by adding SSL and getting it to accept incoming connections for www.example.com and example.com

What I ended up doing, I believe, is creating the equivalent of the Wordpress with NGINX and SSL service from the standard Wordpress service but without the complication of a separate virtual server. There are instructions to do this available on the Bitnami website but they are part of a larger set of instructions and we only need one section https://docs.bitnami.com/tutorials/secure-wordpress-kubernetes-managed-database-ssl-upgrades/ a few simple but key tweaks are needed to get this working on IBM Cloud. 

The first thing to note is that this service is provisioned on top of Kubernetes. That could put you off, especially when we consider that Wordpress cannot take advantage of the scalability offered by Kubernetes without some tinkering. However the process is actually quite straightforward and possibly easier than building Wordpress on your own server. For this reason it's worth considering. And although you'll see the word "cluster" everywhere, that cluster can actually just be one machine.

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
13. Install NFS - [in progress - requird for rest of instructions] 
14. Install WordPress using Bitnami's Helm chart with additional parameters to integrate with Ingress and cert-manager. Replace the YOURDOMAIN placeholder with your domain name. The ciritical difference between this and the Bitnami docs is I have added a parameter to point the correct block storage class (ReadWriteOnce) and correct file storage class (ReadWriteMany) and set the access modes correctly to allow scaling.
```
helm install --set service.type=ClusterIP --set ingress.enabled=true --set ingress.certManager=true --set ingress.annotations."kubernetes\.io/ingress\.class"=nginx --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod --set ingress.hostname=YOURDOMAIN --set ingress.extraTls[0].hosts[0]=YOURDOMAIN --set ingress.extraTls[0].secretName=wordpress.local-tls --set persistence.accessMode=ReadWriteMany --set persistence.storageClass=nfs --set wordpressConfigureCache=true --set memcached.enabled=true --set allowOverrideNone=true --set htaccessPersistenceEnabled=true --set mariadb.primary.persistence.storageClass=ibmc-block-gold wordpress bitnami/wordpress
```
Note: Setting the below options turns on db caching and installs the w3 totalcache plugin using memcached which is highly recomended. See https://github.com/bitnami/charts/tree/master/bitnami/wordpress
```
--set memcached.enabled=true
--set wordpressConfigureCache=true
```

```--set htaccessPersistenceEnabled=true``` 
This makes the .htaccess file persistent and editable by plugins. See here for more into and if you should do this or not https://docs.bitnami.com/kubernetes/apps/wordpress/configuration/understand-htaccess/

Note: All possible settings are described here https://github.com/bitnami/charts/blob/master/bitnami/wordpress/values.yaml

Note: Stuffed up your installation and can't log on etc? Just issue ```helm uninstall wordpress```, delete the Persistant Volume Claim for the mariadb and then issue the comand above again.... that's it!


14. Here we leave the Bitnami documentation. If your DNS update has taken effect, you should be able to connect to your Wordpress instance via your domain name now. You'll notice you have a secure connection.
15. The cert-manager will go off to the Let's Encrypt service and generate SSL certificates for you and then manage them for you! If your DNS update hasn't taken effect yet, cert-manager can't go and make that request and you'll see an extra pod running in your Kubernetes dashboard until it can go and do this.
16. When this all completes you should be able to access Wordpress via your domain name. It may take a while for this complete.
17. The W3 Total Cache plugin will be installed and generally set up. Review the settings and note that memcached is set/available in the list of cacheing servers. 
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
