# Installing Wordpress on IBM Cloud with NGINX and SSL
Wordpress with NGINX and SSL is available in the IBM Cloud catalogue however from experience it may be easier to create an equivelenet Wordpress instance yourself. If you search for Wordpress SSL in the cloud catalogue you'll be presented with a setup page that requires a Virtual Machine cluster and a separate Virtual Machine. That is some significant infrasturcture and may be out of your price range especially if you are just after a production ready Wordpress instance for your business. For you business site you will certainly need SSL.

It should also be noted that the "standard" Wordpress installation doesn't come with any SSL capability. These instructions come from my experince trying to get the standard wordpress installation production ready by adding SSL and getting it to accept incomming connections for www.example.com and example.com

What I ended up doing, I believe, is creating the equivelent of the Wordpress with NGINX and SSL service from the standard Wordpress service but without the complication of a separate virtual server. There are instructions to do this available on the Bitnami website but they are part of a larger set of instructions and we only need one section https://docs.bitnami.com/tutorials/secure-wordpress-kubernetes-managed-database-ssl-upgrades/ a few simple but key tweeks are needed to get this working on IBM Cloud. 

The first thing to note is that this service is provisioned on top of Kubernetes. That could put you off, especially when we consider that Wordpress cannot take advantage of the sclability offered by Kubernetes without some tinkering. However the process is actually quite straightforward and possibly easier than building Wordpress on your own server. For this reason it's worth considering. And although you'll see the word "cluster" everywhere, that cluster can actually just be one machine.

1. Log in to cloud.ibm.com or create your account
2. You'll need a pay as you go account to provision servers so you'll need to upgrade by entering your card details. When you come to create your "cluster" you'll be asked to upgrade there if you haven't aleady done so.
3. Ignore the Wordpress offerings in the catalogue and instead create a Kubernetes cluster
4. Add the IBM Block Storage plugin to your cluster
5. Connect from your command line/terminal to your cluster as described in the Access page of the cluster you created
6. Install helm and the IBM Cloud Command line tool

The following steps 8 to are taken fomr the Bitnami documentation here https://docs.bitnami.com/tutorials/secure-wordpress-kubernetes-managed-database-ssl-upgrades/

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
13. Install WordPress using Bitnami's Helm chart with additional parameters to integrate with Ingress and cert-manager. Replace the DOMAIN placeholder with your domain name. Here I have added a parameter to point the storage class at the correct IBM Storage Class (that was installed by the Block Storage Plugin earlier)
```
helm install --set service.type=ClusterIP --set ingress.enabled=true --set ingress.certManager=true --set ingress.annotations."kubernetes\.io/ingress\.class"=nginx --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod --set ingress.hostname=YOURDOMAIN --set ingress.extraTls[0].hosts[0]=YOURDOMAIN --set ingress.extraTls[0].secretName=wordpress.local-tls --set global.storageClass=ibmc-block-gold --set memcached.enabled=true --set allowOverrideNone=false --set htaccessPersistenceEnabled=true wordpress bitnami/wordpress
```
Note: There are some useful extra options you can add to this hel command
```--set memcached.enabled=true ``` this sets up the w3 Super Cache plugin and would be highly recommeded

These two make the .htaccess file persistent and editiable by plugins. See here for more into and if you should do this or not https://docs.bitnami.com/kubernetes/apps/wordpress/configuration/understand-htaccess/
```--set allowOverrideNone=false ``` 
```--set htaccessPersistenceEnabled=true``` 

Note: Stuffed up your installation and can't log on etc? Just issue ```helm uninstall wordpress```, delete the Persistant Volume Claim for the mariadb and then issue the comand above again.... that's it!


14. Here we leave the Bitnami documentation. If your DNS update has taken effect, you should be able to connect to your wordpress instance via your domain name now. You'll notice you have a secure connection.
15. The cert-manager will go off to the Let's Encrypt service and generate SSL certificates for you and then manage them for you! If your DNS update hasn't taken effect yet, cert-manager can't go and make that request and you'll see an extra pod running in your Kubernetes dashboard until it can go and do this.

Now what about if you want to be able to connect your Wordpress instance using www.mydomain.com as well as mydomain.com. This is a common requirement. Searching for this on the interner will return a lot of scary looking processes but in this case, basically nginx and cert-manager take care of this, you just need to add a few lines to the ingress definition....

[in progress]
