# Kubernetes LEMP Stack
Kubernetes LEMP Stack is a distributed LEMP stack built on top of a Kubernetes cluster. It enables anyone to deploy multiple CMSs (currently WordPress) for any number of websites. We built it to be secure and very fast be default.

Currently this supports Google Compute Engine as a cloud provider. Other providers haven't been tested (things like `PersistentVolume` and `Ingress` depend on your cloud provider).

There are already stable [turn-key deployments for various CMSs](https://github.com/kubernetes/charts "Helm Charts") via Kubernetes Helm Charts, but **Kubernetes LEMP Stack** is designed more or less in the traditional LEMP fashion where you get a bucket for all of your HTML at `/var/www/html` and you may or may not use a CMS.

Actually, **k8s LEMP Stack** should be able to serve as your own personal web server farm! Use it as a backend to your own cloud hosting company! We also want extra customisation in terms of our web server and security hardening measures. In addition, future improvements aim to make this software scalable and highly-available.

## How It Works
* **WordPress**
  * Each WordPress CMS is based on the [wordpress:php7.1-fpm](https://hub.docker.com/r/_/wordpress/ "Official WordPress Docker image") image with extra required PHP extensions such as `redis`. WordPress is contained in one `Deployment` controller along with an NGINX container with FastCGI caching and the NAXSI web application firewall.
  * Each WordPress `Deployment` gets it's own `PersistentVolume` as well as `Secret` objects for storing sensitive information such as passwords for their DBs.
  * `ConfigMap`s are used to inject various `php.ini` settings for PHP 7.1.

* **NGINX**
  * The NGINX container has multiple handy configurations for multi-site and caching, all easily deployed using `ConfigMap` objects.
  * We build the NGINX with the [`nginx-sec-cache`](https://github.com/chepurko/nginx-sec-cache) image, which comes with:
    * [The fork](https://github.com/nginx-modules/ngx_cache_purge) of the [original FRiCLE's module](http://labs.frickle.com/nginx_ngx_cache_purge/)
    * NBS System's [NAXSI module](https://github.com/nbs-system/naxsi). NAXSI means [NGINX](http://nginx.org/) Anti-[XSS](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29) & [SQL Injection](https://www.owasp.org/index.php/SQL_injection).
    * Handy configurations for NGINX and the NAXSI web application firewall are also included via `ConfigMap`s.
  
* **MariaDB**
  * Initially, the WordPress pods all interface with one `mariadb` `StatefulSet`. This is so anyone can start off with a full-fledged web farm and bring up any number of websites using one `mariadb` instance with a databse for each site. Future improvements will allow for HA and scalable clustered RDBMSs.
  * `mariadb` also gets a `PersistentVolume` and `Secret` objects.
  * Updating `StatefulSet` objects in Kubernetes is [currently a manual process](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations), meaning we have to execute MySQL commands in the `mariadb` pod to add new databases and users.
  
* **Redis**
  * To reduce hits to the DB we build the WP image with the `redis` PHP extension and include a Redis `Deployment`.
  * WP must be configured to use Redis upon initialising a new WP site by installing and configuring the WP [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for WordPress") plugin.
  
* **Ingress/Kube Lego**
  * Websites are reached externally via an `nginx` `Ingress` controller. See Kubernetes documentation regarding `Ingress` in the [official docs](https://kubernetes.io/docs/user-guide/ingress/ "Ingress Resources") and on [GitHub](https://github.com/kubernetes/ingress/blob/master/controllers/nginx/README.md "NGINX Ingress Controller").
  * All TLS is terminated at `Ingress` via free Let's Encrypt certificates good for all domains on your cluster. Better yet, certificate issuance is handled automatically with the awesome [Kube Lego](https://github.com/jetstack/kube-lego "Kube Lego").

* See [**Installation and Usage**](USAGE.md) for instructions on getting up and running.
  
![Kubernetes LEMP Stack Architecture](k8s-lemp-stack.png "Kubernetes LEMP Stack Architecture")

## TODO
- [x] Add diagram detailing the general structure of the cluster
- [x] Add working password authentication to Redis
- [ ] High availability
  - [ ] [Ceph distributed storage](https://github.com/ceph/ceph-docker/tree/master/examples/kubernetes "Ceph on Kubernetes")
  - [ ] \(Optional\) HA MySQL via sharding, [clustering](https://thenewstack.io/deploy-highly-available-wordpress-instance-statefulset-kubernetes-1-5/ "Deploy a Highly Available WordPress Instance as a StatefulSet in Kubernetes 1.5"), etc.
  - [ ] Add shared and distributed storage to WordPress deployments so they can then be replicated
- [ ] Enable Drupal CMSs
- [ ] Enable Joomla CMSs
- [ ] Enable generic "HTML" deployments
- [ ] Explore segregating the website deployments in the name of privacy/hardening

## Installation and Usage
Visit the [USAGE.md](USAGE.md)

## Acknowledgements
This project was inspired by the official Kubernetes [WordPress + MySQL sample](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd/ "Persistent Installation of MySQL and WordPress on Kubernetes") and builds on it with the various other official Docker images and Kubernetes applications mentioned previously.
