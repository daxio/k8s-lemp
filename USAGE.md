## Prerequisites
* You need a Kubernetes cluster on Google Compute Engine. This is as easy as following the [official Kubernetes guide](https://kubernetes.io/docs/getting-started-guides/gce/ "Running Kubernetes on Google Compute Engine").
* You should be comfortable with basic SQL statements, i.e. creating and managing DBs, users, grants.
* You also need a domain and access to it's DNS settings. These instructions use the generic domain names www.wingdings.com and www.doodads.com.
* Upon deploying WordPress you should install:
    * [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for WordPress") plugin to connect your site to the Redis `Deployment`
    * A cache-clearing plugin such as [NGINX Cache](https://wordpress.org/plugins/nginx-cache/) if you want to make sure changes appear on your website promptly. There are also other plugins such as [NGINX Helper](https://wordpress.org/plugins/nginx-helper/) but this requires an additional NGINX module and we have not successfully tested this plugin.

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
* Edit the parameters in the StorageClass object in `core-gce-volumes.yaml` to reflect your correct zone and persistent disk type.
* Make sure the disks are in the same `<zone>`as your cluster and that the names match the `pdName` from `core-gce-volumes.yaml`:
  
  ```bash
  $ gcloud compute disks create --size=10GB --zone=<zone> mariadb
  $ kubectl apply -f core-gce-volumes.yaml
  $ kubectl apply -f mariadb-StatefulSet.yaml
  $ kubectl apply -f redis-Deployment.yaml
  ```

### Bring up WordPress/NGINX
* Create `ConfigMap`s
  * First decide what type of WP site you're going to use and rename the corresponding file in `wp/nginx/conf.d/` to `*.conf` and make sure everything else is named `*.conf.OFF`.

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
  > CREATE DATABASE dbWPWD DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  > GRANT ALL PRIVILEGES ON dbWPWD.* TO 'wp-wd'@'%' IDENTIFIED BY 'THE_ACTUAL_PW_FROM_LAST_STEP';
  > FLUSH PRIVILEGES;
  > EXIT;
  ```

* Create the PD for the WordPress/NGINX Deployment.

  ```bash
  $ gcloud compute disks create --size=10GB --zone=<zone> wp-wd
  ```
  
* Deploy WordPress/NGINX and `notls-Ingress`. Change the email address in `lego/kube-lego-Deployment.yaml` before creating the kube-lego Deployment.
 
   __Note: The default domain name is www.wingdings.com, so you should of course change this to your domain in `wp/*tls_Ingress.yaml` files.__
  
  ```bash
  $ kubectl apply -f wp/gce-volume.yaml
  $ kubectl apply -f wp/wp-wd-PVC.yaml
  $ kubectl apply -f wp/wp-wd-Deployment.yaml
  $ kubectl apply -f wp/wp-wd-Service.yaml
  $ kubectl apply -f wp/notls-Ingress.yaml
  $ kubectl apply -f lego/kube-lego-Deployment.yaml
  ```

  * Make sure your site is available at http://www.wingdings.com
  * Finally, enable TLS for you site's Ingress.
  
    ```bash
    $ kubectl apply -f wp-dd/tls-Ingress.yaml
    ```
    
 * Don't forget to install the [Redis Object Cache](https://wordpress.org/plugins/redis-cache/ "Redis Object Cache plugin for WordPress") and [NGINX Helper](https://wordpress.org/plugins/nginx-helper/) plugins if you're going to use Redis and FastCGI cahing, respectively.

## Usage

### Adding a website
* Declare your new website in another directory
  * Make a copy of the `wp/` directory and give it a short name with your website in mind, e.g. `wp-dd/` for "www.doodads.com"

* Update the short name values in your new `wp-dd/*.yaml` files to the corresponding website short name. E.g. `wp-dd`, `wp-dd-pv-claim`, etc.
  ```bash
  $ mv wp-dd/wp-wd-Deployment.yaml wp-dd/wp-dd-Deployment.yaml # do the same for the PVC and Service.yaml
  $ for i in 00-namespace.yaml notls-Ingress.yaml tls-Ingress.yaml gce-volume.yaml \
      wp-dd-PVC.yaml wp-dd-Deployment.yaml wp-dd-Service.yaml; do
      sed -i -r -e 's/wp-wd/wp-dd/' wp-dd/$i
    done
  ```
  
  * Create your new namespace
    ```bash
    $ kubectl apply -f wp-dd/00-namespace.yaml 
    namespace "wp-dd" created
    ```

  * Create `ConfigMap`s
    * First decide what type of WP site you're going to use and rename the corresponding file in `wp-dd/nginx/conf.d/` to `*.conf` and make sure everything else is named `*.conf.OFF`.

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
  > CREATE DATABASE dbWPDD DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
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

* Create another PD.
  ```bash
  $ gcloud compute disks create --size=10GB --zone=<zone> wp-dd
  ```

* Apply the YAMLs for you new site and add the IP address of the NGINX `LoadBalancer` Service you originally created to your domain's DNS settings.
  ```bash
  $ kubectl apply -f wp-dd/gce-volume.yaml
  $ kubectl apply -f wp-dd/wp-dd-PVC.yaml
  $ kubectl apply -f wp-dd/wp-dd-Deployment.yaml
  $ kubectl apply -f wp-dd/wp-dd-Service.yaml
  $ kubectl apply -f wp-dd/notls-Ingress.yaml
  ```
  * Make sure your site is available at http://www.doodads.com
  
  ```bash
  $ kubectl apply -f wp-dd/tls-Ingress.yaml # Enable TLS for your site's Ingress
  ```

### Canary deployments
The best way to push and deploy changes to a production website is with a development or "canary" version which is initially separate from the production site. With WordPress this entails following the [Adding a website](#adding-a-website) section.

* Add an A record in your domain's DNS settings with another name like "dev" pointing to the same Ingress IP address. So the URL of this site would be like https://dev.doodads.com.
* Follow the [Adding a website](#adding-a-website) section and use this new URL for your Ingress resources. This time, use a short name following some versioning scheme. For example `wp-dd-v2` (this is what we will use going forward).
* Make sure to create another database in MariaDB as well, again using your new "wp-dd-v2" short name.
* Once your site is available at https://dev.doodads.com, you'll basically have a fresh WordPress site, just as before.
* Now, the best way we have found to get a replica of your production site into this new "dev" domain is to use a backup or replication plugin. This is cleaner and easier than copying raw files and databases and WordPress handles everything such as search/replacing domain names and putting things in the right place. There are various plugins such as [UpdraftPlus](https://wordpress.org/plugins/updraftplus/). Just use one that can backup/restore and clone/migrate your site to a different domain.
* Now you have two sites, where you can essentially try any new crazy ideas on the *frontend* (themes, design) or *backend* (server/DB configurations) of your canary site until you're happy with it and decide to start **live testing**.

*Frontend* changes can esentially be handled by your backup/cloning plugin between the dev and production sites, but for deploying new *backend* we need to get involved with Kubernetes.

You can, of course, try updating the defintions of the production site to your new specs and utilise an in-place `RollingUpdate` to bring up your new backend, but we're going for the professional route here because, well, Kubernetes makes it fun and easy.

With our awesome Kubernetes LEMP Stack setup we can simply create another `Deployment` with our new "canary" backend behind our current production `Service`. This way we'll have 2 `Deployment`s behind our 1 `Service` which is answering requests on the main https://www.doodads.com domain. The 2 `Deployments` will be hit round-robin until we take down the old version and just leave the new version running. **All without interrupting site traffic**.

* First, start by making sure your dev site is duplicated from your production site and you have your backend configuration setup.
  * **NOTE:** Depending on what you want to acheive, site duplication procedures can vary significantly. For example, if you want to duplicate your production site straight into your canary and begin live testing them round-robin, then you **do not** want to do a search-and-replace on the database for the domain name. You want a pristine copy of the production site dropped into your canary site.
    * If you prefer to work on two separate sites without round-robin, then you will need to do a search-and-replace on our duplicate, and generally follow the [WordPress Codex](https://codex.wordpress.org/Moving_WordPress#Moving_WordPress_Multisite) on moving your site.

* If you're using a Multisite, you will need to modify `wp-config.php` in your canary site. Replace the `xxx`s with your pod name:

  ```bash
  $ kubectl cp wp-dd-v2-xxxxxxxxxx-xxxxx:/var/www/html/wp-config.php ~/wp-config.php
  $ vim ~/wp-config.php
  ```

  * Update the URL in the line `define('DOMAIN_CURRENT_SITE', 'dev.doodads.com');` to your production site URL (www.doodads.com). Then copy the file back to your pod:

  ```bash
  $ kubectl ~/wp-config.php cp wp-dd-v2-xxxxxxxxxx-xxxxx:/var/www/html/wp-config.php
  ```

* Tweak the canary sites's deployment so it's in the same `namespace` and uses the same selector labels as your production site.

  ```bash
  $ cd wp-dd-v2
  ```

  * Update the two `.namespace` fields in the `PersistentVolumeClaim` and `Deployment` definitions to the namespace of your production site (`wp-dd`).
  * Update the `.metadata.labels.app` fields to match you prod. site (`wp-dd`). **Don't change .spec.selector.matchLabels.app**.
  * You should keep the `wp-dd-v2-Service.yaml` file the same as `wp-dd-Service.yaml`, without changing the labels or metadata.
  * Update the two `.metadata.labels.track` fields with value `canary`. You should have something like this:

  ```bash
  ...
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: wp-dd-v2
    namespace: wp-dd
    labels:
      app: wp-dd
      tier: frontend
      track: canary
  spec:
    strategy:
      type: RollingUpdate
    template:
      metadata:
        labels:
          name: wp-nginx
          app: wp-dd
          tier: frontend
          track: canary
  ...
  ```

  * That's all for configuration! Now deploy.

* Start by deleting resources from the dev site's namespace

  ```bash
  $ kubectl --namespace=wp-dev-dd delete deployment wp-dev-dd && \
      kubectl --namespace=wp-dev-dd delete pvc wp-dev-dd-pv-claim && \
      kubectl delete pv wp-dev-dd-pv
  ```

* Create `ConfigMap`s, and `Secret`s in the `wp-dd` namespace.
  * Use the commands from previous sections. Tweak the names if you have to avoid duplicates.
  * You will have to [read out the Secret](https://kubernetes.io/docs/concepts/configuration/secret/#decoding-a-secret) you previously created for you dev site.

* Deploy the canary site and it's PV

  ```bash
  $ kubectl apply -f gce-volume.yaml
  $ kubectl apply -f wp-dd-v2-PVC.yaml
  $ kubectl apply -f wp-dd-v2-Deployment.yaml
  $ kubectl apply -f wp-dd-v2-Service.yaml
  ```

* If all went according to plan, you should now be hitting both, the prod. and canary Deployments at the URL https://www.doodads.com, in a round-robin pattern. Awesome!

* If you want to easily switch between the production and canary sites, all you have to do is add a Selector to your deployed Service:

  ```bash
  $ vim wp-dd-v2-Service.yaml
  ```

  * Add `track: canary` to `spec.selector`, to just hit the canary site, for example. Then `kubectl apply -f wp-dd-v2-Service.yaml` to update the Service configuration.

* When you're satisfied that your canary is stable, delete the old deployment and it's related resources. Something like this is the minimum you would need to delete:

  ```bash
  $ kubectl --namespace=wp-dd delete deployment wp-dd && \
      kubectl --namespace=wp-dd delete pvc wp-dd-pv-claim && \
      kubectl delete pv wp-dd-pv
  ```
