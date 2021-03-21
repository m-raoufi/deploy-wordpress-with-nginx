# **Deploy Wordpress with Nginx**

Deploy wordpress o docker swarm with nginx for the revers proxy server
Running WordPress as a Docker Stack

In the advent of containerized applications it can be quite daunting to get started with docker and multiple containers in a cluster. In this post we will go through some of the terminology and how to get started in a simplified case for a single host, multiple containers application stack, with containers for wordpress, mysql and a reverse proxy using ngnix.


An introduction to Containers
Containers (like Docker) are lightweight processes. They can be used to run application stacks in their own sandboxes. They typically include applications, dependencies, libraries, and config files.

Containers (can be) quite minimal (smaller than a full distribution), as they typically contain just the minimal requirements for your application stack.

Containers are ephemeral. Modifications are lost between container restarts (volumes can be used where persistent data is required).

Containers let you recreate a 'well known' environment. Which can be handy in development teams or production environments where you want your application stack to work the same across different production hosts and developer laptops.

Containers are typically the building blocks of modern clusters that are orchestrated with software like Kubernetes, Docker Swarm, or CoreOS Fleet.

Dockers are an emerging technology. Using them will give you a better understanding of them. And this experience will improve your systems administration knowledge (and CV).

Prerequisites:
Start with a modern distro. Debian 9 works well.

Install Docker version 1.13 or higher.

Creating clusters with docker swarm
A swarm is a group of machines that are running Docker and joined into a cluster. The machines in a swarm can be physical or virtual. After joining a swarm, they are referred to as nodes.

In our example we will be just using a swarm in a single host that it is not part of another bigger, federated network, to init the swarm in the local machine with:

# docker swarm init 
That's it for that! We are not getting into more details on how to join our swam to other networks.

Create a stack of services
Now we will begin setting up a stack, what is a stack you ask? A stack is a group of interrelated services that share dependencies, and can be orchestrated and scaled together. A single stack is capable of defining and coordinating the functionality of an entire application (though very complex applications may want to use multiple stacks).

To define a stack for our example we will use a /root/docker-composer/docker-compose.yml file with:

version: '3'

networks:
   frontend:
   backend:

volumes:
     db_data: {}
     wordpress_data: {}

services:
    db:
      image: mysql:5.7
      volumes:
        - db_data:/var/lib/mysql
      environment:
        MYSQL_RANDOM_ROOT_PASSWORD: '1'
        MYSQL_DATABASE: wordpress
        MYSQL_USER: wordpress
        MYSQL_PASSWORD: wordpressPASS
      networks:
        - backend

    wordpress:
      depends_on:
        - db
      image: wordpress:latest
      volumes:
        - wordpress_data:/var/www/html/wp-content
      environment:
        WORDPRESS_DB_HOST: db:3306
        WORDPRESS_DB_USER: wordpress
        WORDPRESS_DB_PASSWORD: wordpressPASS
        WORDPRESS_DB_NAME: wordpress
      networks:
        - frontend
        - backend

    nginx:
      depends_on:
        - wordpress
        - db
      image: nginx:latest
      volumes:
        - ./nginx.conf:/etc/nginx/nginx.conf
        - /etc/letsencrypt/:/etc/letsencrypt/
      ports:
        - 80:80
        - 443:443
      networks:
       - frontend
Note: You should change wordpressPASS

So, what all this means?, here I will do my best to explain it in layman's terms:

With the network section we are able to define 2 networks, backend and frontend, these networks are private and containers will join them as long each service it is configured to access it (networks parameter in each service). In our example we will use the network backend for the mysql server, the wordpress container will access the backend and frontend network, and the nginx reverse proxy will only operate in the network frontend.

At the volumes section we specify the available volumes to persist data, we hook into these volumes from the services section like for example for the mysql server, where we want to persist the directory /var/lib/mysql, or for wordpress container with the wp-contents directory.

In the services section each container gets specified, with an image to use, volumes to attach, dependencies in other services, environment to use and networks to join. Notice for example how the wordpress service has a reference to the db connection (WORDPRESS_DB_HOST: db:3306) with just the name of the service, or how the the nginx config just uses wordpress for the reverse proxy, dns happens magically!

It is possible to specify files from the file system to be inserted as volumes, like as it is done with the external certificates of letsencrypt or the nginx configuration.

The port settings in the nginx service map the public network port in the host to the container port, this exposes the container service so it can be used externally, eg access wordpress

The nginx configuration file /root/docker-composer/nginx.conf can contain something like this:

events {
 
 }
 
 http {
    error_log /etc/nginx/error_log.log warn;
    client_max_body_size 20m;
    proxy_cache_path /etc/nginx/cache keys_zone=one:500m max_size=1000m;
    server {
      server_name DOMAIN.TLD www.DOMAIN.TLD;
      listen 80;
      listen 443 ssl;
      proxy_cache one;
      ssl_certificate /etc/letsencrypt/live/DOMAIN.TLD/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/DOMAIN.TLD/privkey.pem;
      ssl_protocols TLSv1.1 TLSv1.2;
      ssl_prefer_server_ciphers on;
      ssl_ciphers 
 ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DHE+AES128:!ADH:!AECDH:!MD5;
      location / {
        proxy_pass http://wordpress:80;
         proxy_set_header Host $http_host;
         proxy_set_header X-Forwarded-Host $http_host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header X-Forwarded-Proto $scheme;
 
      }
 
    }
 }
Replace DOMAIN.TLD with your own domain name.

For more options see https://hub.docker.com/_/wordpress

Deploying the stack
Time to deploy our stack, we will name it blog, for this we run:

# docker stack deploy -c /root/docker-composer/docker-compose.yml blog
If all went well we should be able to see our new containers running:

# docker ps
 CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
 8f860c3869df        wordpress:latest    "docker-entrypoint.s…"   7 hours ago         Up 1 minute          80/tcp                blog_wordpress.1.goy8fqjhxq2inqvzu2hi14vso
 029f49732345        nginx:latest        "nginx -g 'daemon of…"   7 hours ago         Up 1 minute          80/tcp                blog_nginx.1.v8w9rl9opa1ap7jv5j2e2gs0s
 d00b09ef5b5b        mysql:5.7           "docker-entrypoint.s…"   7 hours ago         Up 1 minute          3306/tcp, 33060/tcp   blog_db.1.psdhzojctmsi4pdz1yq3b4mlf
To stop all this madness:

# docker stack rm blog
This will stop all the containers and remove any container images, it will leave our volumes untouched.

Note: I included SSL configurations in nginx, the nginx container may not start because of this, there is a chicken and egg paradox here if the certificates files are missing, you can comment out everything ssl related or generate the certificates with certbot as follows:

# certbot certonly --standalone --pre-hook="docker stack rm blog; sleep 30" --post-hook="docker stack deploy -c /root/docker-composer/docker-compose.yml blog" -d DOMAIN.TLD -d 
www.DOMIAN.TLD
Restoring database and website data
So you want the whole shebang with all the trimmings, to restore a db dump into the mysql server container copy the db dump to the db_data volume and restore it, notice that the volumes are named including the blog stack name:

# cp /root/dbDump.sql /var/lib/docker/volumes/blog_db_data/_data/
Access the mysql container, list the id for the container using docker ps first, the following is an example, you will be asked the wordpressPASS with the mysql command.

# docker exec -it d00b09ef5b5b /bin/bash
root@d00b09ef5b5b:/# mysql -uwordpress -p wordpress </var/lib/mysql/dbDump.sql 
root@d00b09ef5b5b:/# rm /var/lib/mysql/dbDump.sql
root@d00b09ef5b5b:/# exit
For the plugins, themes and uploads of wp-contents directory, just restore them over /var/lib/docker/volumes/blog_wordpress_data/_data/, and adjust the permissions.
