## Prerequisites
* You need a Kubernetes cluster on Google Compute Engine. This is as easy as following the [official Kubernetes guide](https://kubernetes.io/docs/getting-started-guides/gce/ "Running Kubernetes on Google Compute Engine").
* You should be comfortable with basic SQL statements, i.e. creating and managing DBs, users, grants.
* You also need a domain and access to it's DNS settings. These instructions use the generic domain names www.wingdings.com and www.doodads.com.
* Upon deploying WordPress you should install the:
    * [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for WordPress") plugin to connect your site to the Redis `Deployment` and the
    * [NGINX Helper](https://wordpress.org/plugins/nginx-helper/) which enables you to clear the FastCGI cache.

## Installation
### Create Namespaces:
  ```bash
  $ kubectl apply -f 00-namespace.yaml
  namespace "core" created
  $ kubectl apply -f wp/00-namespace.yaml 
  namespace "wp-wd" created
  $ kubectl apply -f nginx/00-namespace.yaml 
  namespace "nginx-ingress" created
  $ kubectl apply -f lego/00-namespace.yaml 
  namespace "kube-lego" created
  ```
### Create NGINX Ingress and `default-http-backend` (to catch invalid requests to Ingress and serve 404):
 ```bash
  $ kubectl apply -f nginx/
  namespace "nginx-ingress" configured
  service "default-http-backend" created
  deployment "default-http-backend" created
  service "nginx" created
  configmap "nginx" created
  deployment "nginx" created
  ```
  
* GCE should give you a `LoadBalancer` for your NGINX Service. Watch for your public IP address:
  ```bash
  $ watch kubectl describe svc nginx --namespace nginx-ingress
  ...
  LoadBalancer Ingress:   1.2.3.4
  ...
  ```
* Go to your domain's DNS settings and point your domain to this IP address. After DNS propogates you should see the message "default backend - 404" straight away when visiting your newly set-up domain in a browser. This is the `default-http-backend` doing it's job.

### Create `Secret` objects `mariadb-pass-root`:
  ```bash
  $ openssl rand -base64 20 > /tmp/mariadb-pass-root.txt
  # Delete newlines from the password file or else the password won't work in MariaDB
  $ tr --delete '\n' </tmp/mariadb-pass-root.txt >/tmp/.strippedpassword.txt
  $ mv /tmp/.strippedpassword.txt /tmp/mariadb-pass-root.txt
  $ kubectl create secret generic mariadb-pass-root --from-file=/tmp/mariadb-pass-root.txt --namespace=core
  ```
### Create persistent disks (GCE) and your "core" services:
* Edit the parameters in the StorageClass object in `gce-volumes.yaml` to reflect your correct zone and persistent disk type.
* Make sure the disks are in the same `<zone>`as your cluster and that the names match the `pdName` from `gce-volumes.yaml`:
  ```bash
  $ gcloud compute disks create --size=10GB --zone=<zone> wp-wd
  $ gcloud compute disks create --size=10GB --zone=<zone> mariadb
  $ kubectl apply -f gce-volumes.yaml
  $ kubectl apply -f mariadb-StatefulSet.yaml
  $ kubectl apply -f redis-Deployment.yaml
  ```
### Bring up WordPress/NGINX
* Create `ConfigMap`s

  ```bash
  $ kubectl --namespace=wp-wd create configmap php --from-file=wp/php/conf.d
  $ kubectl --namespace=wp-wd create configmap nginx --from-file=wp/nginx
  $ kubectl --namespace=wp-wd create configmap nginx-conf-d --from-file=wp/nginx/conf.d
  $ kubectl --namespace=wp-wd create configmap nginx-global --from-file=wp/nginx/global
  $ kubectl --namespace=wp-wd create configmap nginx-html --from-file=wp/nginx/html
  ```

* Create a new `Secret` for your new DB user
 
  ```bash
  $ openssl rand -base64 20 > /tmp/mariadb-pass-wp-wd.txt
  $ tr --delete '\n' </tmp/mariadb-pass-wp-wd.txt >/tmp/.strippedpassword.txt
  $ mv /tmp/.strippedpassword.txt /tmp/mariadb-pass-wp-wd.txt
  $ kubectl create secret generic mariadb-pass-wp-wd --from-file=/tmp/mariadb-pass-wp-wd.txt --namespace=wp-wd
  ```

* Manually add a new database in the `mariadb` `StatefulSet` and grant privileges to the WP user.
  ```bash
  $ kubectl --namespace=core exec -it mariadb-0 -- /bin/bash
  root@mariadb-0:/# mysql -u root -p"$MYSQL_ROOT_PASSWORD"
  > CREATE DATABASE dbWPWD DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
  > GRANT ALL PRIVILEGES ON dbWPWD.* TO 'wp-wd'@'%' IDENTIFIED BY 'THE_ACTUAL_PW_FROM_LAST_STEP';
  > FLUSH PRIVILEGES;
  > EXIT;
  ```
  
* Deploy WordPress and `notls-Ingress`. Change the email address in `lego/kube-lego-Deployment.yaml` before creating the kube-lego Deployment.
 
   __Note: The default domain name is www.wingdings.com, so you should of course change this to your domain in `wp/*tls_Ingress.yaml` files.__
  ```bash
  $ kubectl apply -f wp/wp-wd-Deployment.yaml
  $ kubectl apply -f wp/notls-Ingress.yaml
  $ kubectl apply -f lego/kube-lego-Deployment.yaml
  ```

  * Make sure your site is available at http://www.wingdings.com
  
    ```bash
    $ kubectl apply -f wp-dd/tls-Ingress.yaml # Enable TLS for your site's Ingress
    ```

## Usage

### Adding a website
* Declare your new website in another directory
  * Make a copy of the `wp/` directory and give it a short name with your website in mind, e.g. `wp-dd/` for "www.doodads.com"

* Update the short name values in your new `wp-dd/wp-dd-Deployment.yaml` file to the corresponding website short name. E.g. `wp-dd`, `wp-dd-pv-claim`, etc.
  ```bash
  $ mv wp-dd/wp-wd-Deployment.yaml wp-dd/wp-dd-Deployment.yaml # or whatever you want as a short name
  $ for i in 00-namespace.yaml notls-Ingress.yaml tls-Ingress.yaml wp-dd-Deployment.yaml; do
      sed -i -r -e 's/wp-wd/wp-dd/' wp-dd/$i
    done
  ```
  
  * Create your new namespace
    ```bash
    $ kubectl apply -f wp-dd/00-namespace.yaml 
    namespace "wp-dd" created
    ```

  * Create `ConfigMap`s

    ```bash
    $ kubectl --namespace=wp-dd create configmap php --from-file=wp-dd/php/conf.d
    $ kubectl --namespace=wp-dd create configmap nginx --from-file=wp-dd/nginx
    $ kubectl --namespace=wp-dd create configmap nginx-conf-d --from-file=wp-dd/nginx/conf.d
    $ kubectl --namespace=wp-dd create configmap nginx-global --from-file=wp-dd/nginx/global
    $ kubectl --namespace=wp-dd create configmap nginx-html --from-file=wp-dd/nginx/html
    ```
  
  * Create a new `Secret` for your new DB user and save it for the next step
    ```bash
    $ openssl rand -base64 20 > /tmp/mariadb-pass-wp-dd.txt
    $ tr --delete '\n' </tmp/mariadb-pass-wp-dd.txt >/tmp/.strippedpassword.txt
    $ mv /tmp/.strippedpassword.txt /tmp/mariadb-pass-wp-dd.txt
    $ kubectl create secret generic mariadb-pass-wp-dd --from-file=/tmp/mariadb-pass-wp-dd.txt --namespace=wp-dd
    $ cat /tmp/mariadb-pass-wp-dd.txt
    ```
  
  * Again in `wp-dd/wp-dd-Deployment.yaml`, update all `.spec.template.spec.containers[0].env[].value` fields to match your new database name, user, and `secretKeyRef`.

  * Finally update both `.host*:` values in both `wp-dd/*tls-Ingress.yaml` files:

* Manually add a new database in the `mariadb` `StatefulSet` and grant privileges to a new user.
  ```bash
  $ kubectl --namespace=core exec -it mariadb-0 -- /bin/bash
  root@mariadb-0:/# mysql -u root -p"$MYSQL_ROOT_PASSWORD"
  > CREATE DATABASE dbWPDD DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
  > GRANT ALL PRIVILEGES ON dbWPDD.* TO 'wp-dd'@'%' IDENTIFIED BY 'THE_ACTUAL_PW_FROM_LAST_STEP';
  > FLUSH PRIVILEGES;
  > EXIT;
  ```

* Or restore a database from a previous or backed up website:
  ```bash
  $ kubectl cp /path/to/DBbackup/dbWPDD.bak.sql core/mariadb-0:/root/dbWPDD.bak.sql
  $ kubectl --namespace=core exec -it mariadb-0 -- /bin/bash
  root@mariadb-0:/# mysql -u root -p"$MYSQL_ROOT_PASSWORD" dbWPDD < dbWPDD.bak.sql
  ```

* Create another PD and add a new `PersistentVolume` definition into `gce-volumes.yaml` with your corresponding website short name.
  ```bash
  $ vim gce-volumes.yaml # add another PV section changing names to your corresponding short name
  $ gcloud compute disks create --size=10GB --zone=<zone> wp-dd
  $ kubectl apply -f gce-volumes.yaml
  ```

* Apply the YAMLs for you new site and add the IP address of the NGINX `LoadBalancer` Service you originally created to your domain's DNS settings.
  ```bash
  $ kubectl apply -f wp-dd/wp-dd-Deployment.yaml
  $ kubectl apply -f wp-dd/notls-Ingress.yaml
  ```
  * Make sure your site is available at http://www.doodads.com
  
  ```bash
  $ kubectl apply -f wp-dd/tls-Ingress.yaml # Enable TLS for your site's Ingress
  ```