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
  * Updating `StatefulSet` objects in Kubernetes is [currently a manual process](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations), meaning we have to execute MySQL commands in the `mariadb` pod to add new databases and users.
  
* **Redis**
  * To reduce hits to the DB we build the WP/NGINX image with the `redis` PHP extension and include a Redis `Deployment`.
  * WP must be configured to use Redis upon initialising a new WP site by installing and configuring the WP [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for Wordpress") plugin. We protect Redis with a password in a `Secret`.
  
* **Ingress/Kube Lego**
  * Websites are reached externally via an `nginx` `Ingress` controller. See Kubernetes documentation regarding `Ingress` in the [official docs](https://kubernetes.io/docs/user-guide/ingress/ "Ingress Resources") and on [GitHub](https://github.com/kubernetes/ingress/blob/master/controllers/nginx/README.md "NGINX Ingress Controller").
  * All TLS is terminated at `Ingress` via free Let's Encrypt certificates good for all domains on your cluster. Better yet, certificate issuance is handled automatically with the awesome [Kube Lego](https://github.com/jetstack/kube-lego "Kube Lego").

## TODO
- [ ] Add diagram detailing the general structure of the cluster.
- [ ] Configure PHP to listen on a UNIX socket instead of port 9000 and pass the socket file to the `fastcgi_pass` parameter in NGINX.
- [ ] Explore a logging solution for cluster pods such as [Logstash](https://www.elastic.co/guide/en/logstash/current/docker.html "Running Logstash on Docker")
- [ ] High availability
  - [ ] [Ceph distributed storage](https://github.com/ceph/ceph-docker/tree/master/examples/kubernetes "Ceph on Kubernetes")
  - [ ] \(Optional\) HA MySQL via sharding, [clustering](https://thenewstack.io/deploy-highly-available-wordpress-instance-statefulset-kubernetes-1-5/ "Deploy a Highly Available WordPress Instance as a StatefulSet in Kubernetes 1.5"), etc.
  - [ ] Add shared and distributed storage to Wordpress/NGINX deployments so they can then be replicated
 
## Installation
* Create `Secret` objects `mariadb-pass-root` and `redis-pass`.
  ```bash
  $ openssl rand -base64 20 > /tmp/mariadb-pass-root.txt
  $ openssl rand -base64 20 > /tmp/redis-pass.txt
  $ kubectl create secret generic mariadb-pass-root --from-file=/tmp/mariadb-pass-root.txt
  $ kubectl create secret generic redis-pass --from-file=/tmp/redis-pass.txt
  ```

* Create a `ConfigMap` for `nginx`
  ```bash
  $ kubectl create configmap nginx-config --from-file=configMaps/nginx/
  ```

## Usage

### Adding a website
* Manually add a new database in the `mariadb` `StatefulSet` and grant privileges to a new user.
  ```bash
  $ commands
  ```
* Create a new `Secret` for your new DB user
  ```bash
  $ commands
  ```

* Declare your new website in another YAML file
  * Make a new copy of the `wp-mywebsite.yaml` file
  * Update the following values in your new `wp-mywebsite-2.yaml` file to the corresponding website name of your choosing. E.g. `wp-mywebsite-2`, `wp-mywebsite-2-pv-claim`, etc.
    * `.metadata.name`
    * `.metadata.labels.app`
    * `Service` definition
      * `.spec.selector.app`
    * `PersistentVolumeClaim` definition
      * `.spec.selector.matchLabels.app`
    * `Deployment` definition
      * `.spec.template.metadata.labels.app`
      * Update all `.spec.template.spec.containers[0].env[].value` fields to match your new database name, user, and password from `Secret`
      * `.volumes[0].persistentVolumeClaim.claimName`

## Acknowledgements
This project is based on the official Kubernetes [Wordpress + MySQL sample](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd/ "Persistent Installation of MySQL and WordPress on Kubernetes") and builds on it with the various other official Docker images and Kubernetes deployments mentioned previously.
