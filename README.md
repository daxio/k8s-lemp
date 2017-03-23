# Kubernetes LEMP Stack
Kubernetes LEMP stack is a distributed LEMP stack built on top of a Kubernetes cluster. It enables anyone to deploy multiple CMSs (currently Wordpress) for any number of websites.

Currently this supports Google Compute Engine as a cloud provider. Other providers haven't been tested (things like `PersistentVolume` and `Ingress` depend on your cloud provider).

## How It Works
* **Wordpress/NGINX**
  * Each Wordpress CMS is based on the [wordpress:php7.1-fpm](https://hub.docker.com/r/_/wordpress/ "Official Wordpress Docker image") image with extra required PHP extensions such as `redis`. Wordpress is contained in one `Deployment` controller along with an `nginx` container. The `nginx` image is planned to include build-time security enhancements such as the [NAXSI WAF](https://github.com/nbs-system/naxsi "NBS System NAXSI Web Application Firewall").
  * Each Wordpress/NGINX `Deployment` gets it's own `PersistentVolume` as well as `Secret` objects for storing sensitive information such as passwords for their DBs.
  
* **MariaDB**
  * Initially, the Wordpress/NGINX pods all interface with one `mariadb` `StatefulSet`. This is so anyone can start off with a full-fledged web farm and bring up any number of websites using one `mariadb` instance with a databse for each site. Future improvements will allow for HA and scalable clustered DBs.
  * `mariadb` also gets a `PersistentVolume` and `Secret` objects.
  
* **Redis**
  * To reduce hits to the DB we build the WP/NGINX image with the `redis` PHP extension and include a Redis `Deployment`.
  * WP must be configured to use Redis upon initialising a new WP site by installing and configuring the WP [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for Wordpress") plugin. We protect Redis with a password in a `Secret`.
  
* **Ingress/Kube Lego**
  * Websites are reached externally via an `nginx` `Ingress` controller. See Kubernetes documentation regarding `Ingress` in the [official docs](https://kubernetes.io/docs/user-guide/ingress/ "Ingress Resources") and on [GitHub](https://github.com/kubernetes/ingress/blob/master/controllers/nginx/README.md "NGINX Ingress Controller").
  * All TLS is terminated at `Ingress` via free Let's Encrypt certificates good for all domains on your cluster. Better yet, certificate issuance is handled automatically with the awesome [Kube Lego](https://github.com/jetstack/kube-lego "Kube Lego").

## TODO
Add diagram detailing the general structure of the cluster.

## Acknowledgements
This project is based on the official Kubernetes [Wordpress + MySQL sample](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd/ "Persistent Installation of MySQL and WordPress on Kubernetes") and builds on it with the various other official Docker images and Kubernetes deployments mentioned previously.
