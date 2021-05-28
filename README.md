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


There are two ways to install this on IBM Cloud.

1. Using Classic infrastructure [here](https://github.com/tim-minter/wordpress-on-ibm-cloud/blob/main/on-classic-infrastructure.md)
2. Using Virtual Private Cloud infrastructure (recommended) [here](https://github.com/tim-minter/wordpress-on-ibm-cloud/blob/main/on-virtual-private-cloud-infrastructure.md)

The second option is recommended but I'm still working on these intructions as of 28/5/21. Currently they work but I havn't ironed out how to to connect into the wordpress instance via the load balancer.
