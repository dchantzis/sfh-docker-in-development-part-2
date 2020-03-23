# Servers For Hackers - Docker in Development Part I
Source: https://serversforhackers.com/s/docker-in-dev-v2-i

We will build an application Image and from that we can start and run Containers
that will run our application. We will see how we can connect our application Container
to a mysql Container and make sure they are connected to a common network so that they can talk to each other over a network.
And in this way we will run an application, run migrations and have a database to save to,
and how to persist that data using Docker Volumes.
We will make an Image using a Dockerfile.

## 01: Course Intro
Source: https://serversforhackers.com/c/div-intro

## 02: Images vs Containers
Source: https://serversforhackers.com/c/div-images-vs-containers

### Images
A docker Image is like a Class. A Container is an instance of an Image, just like an Object is an instance of some Class.

To list images:
```
docker image ls
```

### Running a Container

To run a Container from an image:
```
docker run --rm -d -v $(pwd):/application:/var/www/html -p 8080:80 shippingdocker/phpapp
```
Options
- --rm destroy the container once we finish
- -d run in the background
- -v to share a directory, example: $(pwd):/application:/var/www/html put the directory "application" in "/var/www/html"
- -p share port 8080 to port 80 inside the container
- shippingdocker/phpapp: run an image with this name

In a project to start the Containers (in the background):
```
docker-compose up -d
```

In a project to stop the running Containers:
```
docker-compose down
```

To check all the running Containers:
```
docker ps
```

To stop a specific Container:
```
docker stop [CONTAINER_ID]
```

To remove a specific Container:
```
docker rm [CONTAINER_ID]
```

To list images:
```
docker image ls
```

To list only the Image ID column of the running images:
```
docker image ls -q
```

To remove all running Images:
```
docker image rm $(docker image ls -q)
```

---

## 03: Docker Images

### Goals

1. Run a dev environment in Docker, with the following containers:
	- Nginx/PHP
	- MySQL
	- Redis
	Every Container from the above 3, will be in its own separate directory upder ./docker/app/
2. Build a `Dockerfile` to build an nginx/php image
3. Since we're not customising the installation for MySQL and Redis, we can use the official images for MySQL/Redis.

Create a new project directory, named "php-app":
```
mkdir php-api
cd php-api
mkdir docker
mkdir docker/app
```

Create the `Dockerfile`:
```
cd docker/app
vim Dockerfile
```

Add the following (to find the latest official ubuntu releases, go to https://hub.docker.com/_/ubuntu)
```
FROM ubuntu:18.04
```

Exit vim and run:
```
docker run --rm -it ubuntu:18.04 bash
```
Options:
- --rm - remove it once stop running it
- -it - make is interactive 
- ubuntu:18:04 - the instance we are running
- bash - to make it look like we are inside the container

Then run:
```
apt-get update
apt-get install -y nginx
```

To check if nginx was installed:
```
which nginx
```
Then we can inspect the Container to see the changes.
Open a new terminal window and check the running processes (from outside the Container):
```
docker ps
```
get the Container ID of the running process
and then:
```
docker diff [CONTAINER_ID]
```
This shows all the changes to the container (files added, modified, deleted).
We can then commit thos echanges to make a new Image from that Container and its changes:
```
docker commit -a "Dimitris Hantzis" -m "Installed Nginx" [CONTAINER_ID] [REPOSITORY_NAME:TAG]
```
This way we commit changes made to the running container:
- -a Author string
- -m The commit message
- [CONTAINER_ID] We give it the container from which to make a new image
- [REPOSITORY_NAME:TAG] We give the image a name of mynginx and tag is latest, example: mynginx:latest

The commit has created a new image with the changes from the running Container
Check the current Images:
```
docker ls
```
Now instead of creating a new image we can run the newly-created one, like so:
```
docker run --rm -it mynginx:latest bash
```
And that is the process of what happens inside of a `Dockerfile`.
For every change in the `Dockerfile` -- every added line -- it will create a
new Container, make a change, commit that change and make a new Image off of it.
At the end we get a final Image that we can use. 

Every intermediary container or image 
that gets created, will get destroyed automatically, it gets cleaned out.
We just get the last image of the completed work as a result.

---

## 04: Using Dockerfiles
Source: https://serversforhackers.com/c/div-docker-images

We use a Dockerfile to build a new image. Dockerfiles make creating an image from 
a container easier (it's code in a file you can commit SCM).

```
vim Dockerfile
```

For every change, every instruction that we write, will create a new Image, do the changes and then commits that change.
For this reason it is a good practice to have the least amount of instructions, as it makes the process quicker.

Here's the Dockerfile we end up with:
```
FROM ubuntu:18.04

LABEL maintainer="Chris Fidao"

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y gnupg tzdata \
    && echo "UTC" > /etc/timezone \
    && dpkg-reconfigure -f noninteractive tzdata

RUN apt-get update \
    && apt-get install -y curl zip unzip git supervisor sqlite3 \
       nginx php7.2-fpm php7.2-cli \
       php7.2-pgsql php7.2-sqlite3 php7.2-gd \
       php7.2-curl php7.2-memcached \
       php7.2-imap php7.2-mysql php7.2-mbstring \
       php7.2-xml php7.2-zip php7.2-bcmath php7.2-soap \
       php7.2-intl php7.2-readline php7.2-xdebug \
       php-msgpack php-igbinary \
    && php -r "readfile('http://getcomposer.org/installer');" | php -- --install-dir=/usr/bin/ --filename=composer \
    && mkdir /run/php \
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

We can now remove the unused image:
```
docker image ls
docker image rm mynginx:latest
```

We can build the new image like so:
```
cd php-app/docker/app
docker build -t shippingdocker/app:latest -f ./Dockerfile .
```
Some notes regarding the above command:
- -t shippingdocker/app:latest The image name and tag
- -f .Dockerfile
- `.` The context of where everything is, where you want to run stuff. That is not
necessarily the directory where the Dockerfile is in, but the directory anything that is
referenced in the Dockerfile is relative to. So if the Dockerfile references
any files on your local filesystem to pull in the docker image that it makes, that is going to be relative
to whatever directory you reference as the context, and that context is the current directory.

For example (first go to the root directory of the project):
```
cd ./../../
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
```
```
docker image ls
```
We can run a Container based on the Image we have just created:
```
docker run --rm -it shippingdocker/app:latest bash
```

To show the currently running Containers (processes)
```
docker ps
```
To show all containers:
```
docker ps -a
```

To stop nginx from running in the background of docker, we need to add the following in the `Dockerfile`:
```
&& echo "daemon off;" >> /etc/nginx/nginx.conf
```
Then re-build the docker image:
```
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
```
Check the images
```
docker image ls
```
To run the Image and push it to the backround we remove the flag -it and add -d, like so:
```
docker run -d --rm shippingdocker/app:latest nginx
```
nginx is now runn on the foreground. To check, run the following:
```
docker ps
```

---

## 05: Serving Web Files
Source: https://serversforhackers.com/c/div-serving-web-files

Remove the running Container:
```
docker ps -a
docker stop <CONTAINER_ID> && docker rm <CONTAINER_ID>
```

We are going to edit the shippingdocker/app image and add some configuration to it.
The configuration will be added in the `docker/app` directory in the `default` file:
``` 
vim docker/app/default
```
add the following default configuration:
```
server {
	listen 80 default_server;

	root /var/www/html/public;

	index index.html index.htm index.php;

	server_name _;

	charset utf-8;

	location = /favicon.ico { log_not_found off; access_log off; }
	location = /robots.txt { log_not_found off; access_log off; }

	location / {
		try_files $uri $uri/ /index.php$is_args;$args;
	}

	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/run/php/php7.2-fpm.sock;
	}

	error_page 404 /index.php;
}
```

In the `Dockerfile` we'll write the following to add the file for nginx to use the `default` configuration:
```
ADD default /etc/nginx/sites-available/default
```
To rebuild the image run the following below. This over-writes our current image at tag latest with our changes:
```
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
```
Add some files in `docker/app`:
```
mkdir application
vim application/index.php
vim application/index.html
```
Contents for `application/index.php`:
```
<?php phpinfo();
```
Contents for `application/index.html`:
```
Hello Docker.
```

Now run Container:
```
docker run --rm -d -v $(pwd)/application:/var/www/html/public -p 8080:80 shippingdocker/app:latest nginx
```
- -p 8080:80 This will forward the port 8080 of my localhost to the port 80 inside of the Container.
- -v Share a volume. We are sharing the current directory (`$(pwd)`) and share the application directory inside there `(application)`
to the directory inside the Container (`/var/www/html/public`) were we set the web-root inside the Container.

*Note* for the Windows Command Line, you can mount the current directory like so:
```
-v %cd%/application:/var/www/html/public
```
For Powershell, use ${PWD}:
```
-v ${PWD}/application:/var/www/html/public
```
For Linux, use $(pwd):
```
-v $(pwd)/application:/var/ww/html/public
```

With the Container running on port 8080, we can now check the configuration, (or visit localhost:8080 from the browser):
```
curl localhost:8080
```

From the browser, running `localhost:8080/index.php` will result a 502 error.
This is because we only told the Container to run `nginx`
If we do `docker exec` we will execute against an already-running Container (previously done with `docker run`).
We can execute (interactively) `bash` inside the already-running Container:
```
docker exec -it <CONTAINER_ID> bash
```
then to see all the running processes inside the Container:
```
ps aux
```
This will show that we are not running php (thus the 502 error above).
Php FPM needs to be running to fulfill a php request and we have only specified to run nginx.

---

## 06: Running Multiple Processes
Source: https://serversforhackers.com/c/div-running-multiple-processes
We need to get both PH and Nginx running on this Container at the same time.
We'll use Supervisord to do that.

Supervisor is a process monitor. It's capable of starting multiple processes and making sure they stay alive.
So we're going to be using this to make sure we have nginx and php fpm running at the same time when the Container starts.

We'll create a new configuration file for supdervisor, named `supervisord.conf` to specify the running programs to monitor.
```
vim docker/app/supervisord.conf
```
and add the following lines below. Supervisor should not "daemonize" itself and run in the foreground,
because the Container will run supervisor and Docker needs to see that process in the foreground.

Add the following lines in `docker/app/supervisord.conf`:
```
[supervisord]
nodaemon=true

[program:nginx]
command=nginx
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:php-fpm]
command=php-fpm7.2
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

Then add the following inside `docker/app/Dockerfile`:
```
ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf
```

Rebuild the Image:
```
docker build -t shippingdocker/app:latest -f docker\app\Dockerfile docker\app
```

Run the Container with supervisord:
```
docker run --rm -d -v %cd%/application:/var/www/html/public -p 8080:80 shippingdocker/app:latest supervisord
```

Execute a command (`bash`) in the running Container
```
docker exec -it <CONTAINER_ID> bash
```
and check supervisor is running (inside the container):
```
ps aux
```
from inside the container we can check (using port 80):
```
curl localhost:80/index.php
```
from outside the container we can check (using port 8080):
```
curl localhost:8080/index.php
```

In the `Dockerfile` we can specify a command to run by default (if no other command is given), by adding the following line:
```
CMD ["supervisord"]
```
then we rebuild the Image:
```
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
```
then we run the Container without specifying a process:
```
docker run -d -p 8080:80 -v %cd%/application:/var/www/html/public --rm shippingdocker/app:latest
```
and check if everything works fine (to get the <CONTAINER_ID>, run `docker ps`:
```
docker exec -it <CONTAINER_ID> basha
```
and inside the container:
```
ps aux
```
```
curl localhost:80/index.php
```

---

## 07. Configuring PHP-FPM
Source: https://serversforhackers.com/c/div-configuring-php-fpm

We need to add some custom PHP configuration, especially so it does not run as a daemon.

```
> docker run -d --rm -p 8080:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest
> docker exec -it <CONTAINER_ID> bash
> cd /etc/php/7.2/fpm
> cat php-fpm.conf | grep daemon
> exit
```
Now we will copy the `php-fpm.conf` from inside the Container to our local `docker/app` directory:
```
> docker exec -it <CONTAINER_ID> cat /etc/php/7.2/fpm/php-fpm.conf
```
Then we will update the `php-fpm.conf` file by setting `daemonize = no`
and add the following line in the `Dockerfile`:
```
ADD php-fpm.conf /etc/php/7.2/fpm/php-fpm.conf
```

We will need to rebuild the Image and run the Container:
```
> docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
> docker run -it --rm -p 8080:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest
```

We will se in the output that php-fpm is running:
```
2020-03-18 09:28:55,294 INFO spawned: 'nginx' with pid 8
2020-03-18 09:28:55,295 INFO spawned: 'php-fpm' with pid 9
2020-03-18 09:28:56,330 INFO success: nginx entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2020-03-18 09:28:56,331 INFO success: php-fpm entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```
## 07: Docker Logs
Source: https://serversforhackers.com/c/div-docker-logs

We need to make a tweak so everything outputs to stdout or stdeer.
If we do that successfully, we can use Docker's logging mechanism to see 
output form Nginx, PHP and Supervisord.

In a previous section we defined in `docker/app/supervisor.conf` where the stdout and stderr will output for nginx and php:
```
[program:nginx]
...
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr
...
[program:php-fpm]
...
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr
```

We can see that output (for a running Container), like so:
```
> docker run -d --rm -p 8080:80 -v %cd%/appliation:/var/www/html/public shippingdocker/app:latest

> docker logs <CONTAINER_ID> -f
```
- -f Optional parameter to follow the log on the command line (live output on the command line).

However, we have output from supervisor but not from nginx or php.

Nginx
---
We can adjust the Dockerfile by symlinking the nginx error and access
logs to stdout and stderr. We will add the symlink in the `Dockerfile`:
```
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
```

PHP-FPM
---
Then in our `php-fpm.conf` file adjust the error_log to:
```
error_log = /proc/self/fd/2
```

Re-build the container and run interactively:
```
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
docker run -d --rm -p 8080:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest
docker logs <CONTAINER_ID> -f
```

Then refresh the browser to `localhost:8080/index.php` and check the output on the console.

---

## 08: Entrypoint vs Cmd
Source: https://serversforhackers.com/c/div-entrypoint-vs-cmd

We see how to use an ENTRYPOINT instead of CMD within a Dockerfile in order to setup a script
that is always run (no matter what) when a container is spun up from the image we make.

Create the new file `start-container.sh` inside `docker/app`:
```
vim docker/app/start-container.sh
```

Update the `Dockerfile` by adding the following (remove or comment out the line `CMD ["supervisord"]`):
```
ADD start-container.sh /usr/bin/start-container.sh
RUN chmod +x /usr/bin/start-container

ENTRYPOINT ["start-container"]
```

First make sure `env` exists in the Container:
```
docker run -it --rm -p 8080:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest bash

which env
```
so `> env bash` will give us the correct bash.

Copy the location of the `env` file (example: `/usr/bin/env`).

Update the `start-container.sh`:
```
#!/usr/bin/env bash

##
# Ensure /.composer exists and is writable
#
if [ ! -d /.composer ]; then
    mkdir /.composer
fi

chmod -R ugo+rw /.composer

##
# Run a command or start supervisord (default to supervisord)
#
if [ $# -gt 0 ];then
    # If we passed a command, run it
    exec "$@"
else
    # Otherwise start supervisord
    /usr/bin/supervisord
fi
```

This script allows us to do some pre-processing whenever a container is spun up.
It defaults to running supervisord for us, but we also allow ourselves to run any command,
just like we could before, by using the exec "$@" line if we pass any commands/arguments to the container.

Re-build the Image and run the Container:
```
docker build -t shippingdocker/app:latest -f docker/app/Dockerfile docker/app
docker run -d --rm -p 8080:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest
```

---

## 10: Docker Networks Intro
Source: https://serversforhackers.com/c/div-docker-networks-intro

We will see how we can get containers to talk to each other over a Docker network.

---
We will create two containers and add them into the new network appnet.

The containers then can reference eachother - the hostname for each container is taken from the --name
given to the container. This lets the app container talk to mysql over hostname `mysql`.
Within the container, you can run `getent hosts mysql` to see the hostname resolution of hostname `mysql`
---

To list the default Docker networks:
```
docker network ls
```

Below we will create a new network and create two additional containers and make them communicate with each other.
```
docker network create appnet

docker network ls
```

We will create a MySQL Container and hook it up with the application container so that they can communicate with each other:

We will go to `https://hub.docker.com/_/mysql` and use the MySQL 5.7

```
docker run --rm -d --name=mysql --network=appnet -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=homestead -e MYSQL_USER=homestead -e MYSQL_USER_PASSWORD=secret mysql:5.7
```
- --name The name of the host

We can check everything:
```
docker image ls

docker network ls

docker network inspect appnet
```

The last command will show us the containers inside the `appnet` network.
We can check the IPv4 Address used for the host that we named `mysql`

Next:
```
docker exec -it mysql bash

which mysql

getent hosts mysql

mysql -h mysql -u root -p
```
- -h Is the host that we named `mysql` inside the container

Now we will create the second Container under the same network `appnet`:
```
docker run --name=app --network=appnet -d --rm -p 80:80 -v %cd%/application:/var/www/html/public shippingdocker/app:latest 
```
To see them both running (`mysql` and `app`):
```
docker ps
```
Bash into `app`:
```
docker exec -it app bash
```
Then check the IP address of the host `app`:
```
getent hosts app
```
Check the IP address of the host `mysql` (this means they can see each other):
```
getent hosts mysql
```
Exit the container and inspect the network `appnet` to view both of the Containers of the network:
```
docker network inspect appnet
```

## 11: Connecting Containers
Source: https://serversforhackers.com/c/div-connecting-containers

We get a Laravel application running.

Stop the running Containers of the application `shippingdocker/app:latest` 
```
docker ps ls

docker stop <CONTAINER_ID>
```

We will remove the `application` directory:
```
rm -rf application
```

We will "share" the root directory to be mapped in the container, as we'll install laravel in a new `application` directory in the root folder:
```
docker run -it --rm -v %cd%:/var/www/html -w /var/www/html shippingdocker/app:latest composer create-project laravel/laravel application 
```
- -w Working directory inside the Container.
We specify a working directory so that the Container knows where to install laravael.

Now that laravel is installed in the `application` directory, we will map `application`. Please note that by default
laravel has a `public` directory:

```
docker run --name=app --network=appnet -d --rm -p 80:80 -v %cd%/application:/var/www/html shippingdocker/app:latest
```
- -p We set the port to 80:80 (<local_machine_port>:<container_port) instead of 8080 as it doesn't matter for this test, as long as the local port 80 is free.

```
docker exec -it -w /var/www/html app php -v

docker exec -it -w /var/www/html app php artisan -v
```

As of Laravel 6.x (and 7.x) the authentication scaffolding is no longer available with `php artisan make:auth`
We will need to install the `laravel/ui` package via composer:

So instead of (usable up to laravel 5.8):
```
docker exec -it -w /var/www/html app php artisan make:auth
```
we need to do the following:
```
docker exec -it -w /var/www/html app composer require laravel/ui

docker exec -it -w /var/www/html app php artisan ui:auth
```
and then run the migrations (the auth scaffolding requires it):
```
docker exec -it -w /var/www/html app php artisan migrate
```
if the migrations fail, that means we need to update the projects' `.env` file (inside the `application` local project root folder):
```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=root
DB_PASSWORD=root
```
Notice that for `DB_HOST` we use the hostname for the container named `mysql` that we created earlier.

Now we can try again:
```
docker exec -it -w /var/www/html app php artisan migrate
```
Go to `localhost:80` check your installed Laravel project, then register, sign-out and log in.

Now we can also check our network:
```
docker network inspect appnet
```

---

## 12: Docker Volumes
Source: https://serversforhackers.com/c/div-docker-volumes

We will cover Docker volumes and make sure we have set our docker-compose.yml file correctly to use our created volumes.

The following command will list the existing Volumes:
```
docker volume ls
```

The listed Volume is from the `mysql` Container as it was created using a Dockerfile that is listing a Volume.
We can view the used Dockerfile at [https://github.com/docker-library/mysql/blob/d284e15821ac64b6eda1b146775bf4b6f4844077/5.7/Dockerfile](https://github.com/docker-library/mysql/blob/d284e15821ac64b6eda1b146775bf4b6f4844077/5.7/Dockerfile):
```
FROM debian:buster-slim
...
.
.
.
...
VOLUME /var/lib/mysql
...
```
This will tell Docker to create a Volume and share it in the `/var/lib/mysql` directory in the Container.

---

We can kill our `mysql` Container:
```
docker kill <CONTAINER_ID>
```
and check that it's gone:
```
docker ps -a
```
If we check our Volumes we will see that the previous Volume is gone and with it our database data:
```
docker volume ls
```
We can now create a new volume like so:
```
docker volume create mydata
```
and now we can:
```
docker run -d --rm --name=mysql -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=homestead -e MYSQL_USER=homestead -e MYSQL_PASSWORD=secret --network=appnet -v mydata:/var/lib/mysql mysql:5.7
```
- -v Specify the location to share the `mydata` Volume within our Container and bind it to "/var/lib/mysql" : `-v mydata:/var/lib/mysql`

We can now inspect the `mysql` Container:
```
docker inspect mysql
```
and we can see that under "Mounts" we have a Volume named "mydata" mounted to that Container successfully.

---

Now we need to run the migrations again in the `app` Container (since we lost all the data when we initially killed the previous `mydata` Container and then recreated it):
```
docker exec -it -w /var/www/html app php artisan migrate
```

We can now run the following to view all the database tables of the database name "homestead" inside the `mysql` Container,
by connecting to mysql (using the credentials with -u and -p) and executing a command (with -e):
```
docker exec -it mysql mysql -u root -p -e "USE homestead; SHOW TABLES;"
```

Now we can kill the running Container `mysql`:
```
docker kill <CONTAINER_ID>
```
and check our Volumes:
```
docker volume ls
```
and we can see that our Volume `mydata` still exists.

Now we can run ("spin out") a totally new instance of `mysql` and connect it to the same existing Volume `mydata` with our database data.

```
docker run -d --name=mysql --network=appnet -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=homestead -e MYSQL_USER=homestead -e MYSQL_PASSWORD=secret --network=appnet -v mydata:/var/lib/mysql mysql:5.7
```
Don't forget to specify the following:
- --name The hostnname 
- --network The network

Now we can see our database data (that were created from the laravel migrations), like so:
```
docker exec -it mysql mysql -u root -p -e "USE homestead; SHOW TABLES;"
```

We can create and destroy MySQL containers as much as we want. As long as we share the `mydata` volume,
the database data will persist (assuming, of course, we don't delete the volume via `docker volume rm mydata`!)

---

# Docker in Development Part II
Source: https://serversforhackers.com/s/docker-in-dev-v2-ii

We continue the Docker in Dev course by using Docker Compose and building a development workflow.

## 01: Docker Compose Intro

We will see how we can start to use Docker Compose to more easily control our Docker environment.

We'll start by building a `docker-compose.yml` file so we can manage our Docker environment more easily.

Docker Compose lets us manage the life cycle of our Docker environment, including Volumes, Networks, and of course
helping us manage multiple containers that can work together.

We'll start defining some simple things in our `docker-compose.yml` file:
```
vim docker-compose.yml
```
```
version: '3'
services:
networks:
  appnet:
    driver: bridge
volumes:
  dbdata:
    driver: local
  cachedata:
    driver: local
```

## 02: Docker Compose Services
Source: https://serversforhackers.com/c/div-docker-compose-services

We fill out a few services into our `docker-compose.yml` file.

---

Let's delete all our existing Volumes:
```
docker volume rm $(docker volume ls -q)
```
or individually:
```
docker volume rm <VOLUME_NAME>
```

We'll do the same for the networks, but delete the default ones:
```
docker network rm appnet
```
If it fails with an error like the following below, then we need to stop the running processes (containers) and then try again:
```
Error response from daemon: error while removing network: network appnet id 9e1469bfde3b5a8bf2f2de6fb0566b39aea4dc871d4dd109da0b747d2ea48585 has active endpoints
```
```
docker stop <CONTAINER_ID>
```

We will update the docker-compose.yml, like so:
```
version: '3'
services:
  cache:
    image: redis:alpine
    networks:
     - appnet
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: homestead
      MYSQL_USER: homestead
      MYSQL_PASSWORD: secret
    networks:
     - appnet
networks:
  appnet:
    driver: bridge
volumes:
  dbdata:
    driver: local
  cachedata:
    driver: local
```

Then:
```
docker-compose ps
```
```
docker-compose up -d
```

We can check everything with the following:
```
docker image ls
docker ps
docker network ls
docker volume ls
```
We notice that all the names are prefixed with the root directory name `php-api_`.

We can now:
```
docker-compose down

docker ps -a

docker volume ls
```
This command doesn't only stop the Containers but also destroys them.
It removes the Network, but it doesn't remove the Volume.

If we bring the Containers up again they will be using the existing Volumes:
```
docker-compose up -d
```

Instead of doing a `dockler-compose down`, we could be doing `start` or `stop`, however this will keep the Containers running.

## 03: Compose and Volumes
Source: https://serversforhackers.com/s/docker-in-dev-v2-ii

If we check our volumes:
```
docker volume ls
```
we notice 2 new Volumes getting created every time the Containers run. These additional volumes
are created by the cachedata and dbdata because in our docker-compose.yml we did not specify
which volumes they should be using.

The resulting docker-compose.yml file should now look like this:
```
version: '3'
services:
  cache:
    image: redis:alpine
    networks:
     - appnet
    volumes:
     - cachedata:/data
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: homestead
      MYSQL_USER: homestead
      MYSQL_PASSWORD: secret
    networks:
     - appnet
    volumes:
     - dbdata:/var/lib/mysql
networks:
  appnet:
    driver: bridge
volumes:
  dbdata:
    driver: local
  cachedata:
    driver: local
```

We need to remove all current Volumes and start the containers again:
```
docker volume rm $(docker volume ls -q)

docker-compose up -d

docker volume ls
```

Then check everything is getting used properly and consistently:
```
docker-compose down && docker-compose up -d && docker volume ls
```

---

## 04: Our App Service
Source: https://serversforhackers.com/c/div-our-app-service

We fill out the `app` service within our Docker Compose configuration file.

---

We need to update the docker-compose.yml file and add our application service, like so:

```
version: '3'
services:
        app:
                build:
                        context: ./docker/app
                        dockerfile: ./docker/app/Dockerfile
                image: shippingdocker/app:latest
                networks:
                        - appnet
                volumes:
                        - ./application:/var/www/html
                ports:
                        - 80:80
```

then:
```
docker-compose down && docker-compose up -d
```

We also need to make the following updates to the applications .env file:
```
vim ./application/.env
```
```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=root
DB_PASSWORD=root
```
```
REDIS_HOST=cache
REDIS_PASSWORD=null
REDIS_PORT=6379
```
```
BROADCAST_DRIVER=log
CACHE_DRIVER=redis
QUEUE_CONNECTION=sync
SESSION_DRIVER=redis
SESSION_LIFETIME=120
```

Now we need to install redis via composer in our applications' Container:
```
docker-compose exec -w /var/www/html app composer require predis/predis
```
The above will fail, so we can run the following:
```
docker-compose exec app bash -c "cd /var/www/html && composer require predis/predis"
```
We need to run the migrations again for our application:
```
docker-compose exec app bash -c "cd /var/www/html && php artisan migrate"
```
We can check our application via the browser at localhost:80, or:
```
curl localhost:80
```
We now can register and then login to our application successfully.

---

## 05: The Working Directory

We will update our docker-compose.yml and add the working directory for our application `app`:
```
version: '3'
services:
        app:
                build:
                        context: ./docker/app
                        dockerfile: ./docker/app/Dockerfile
                image: shippingdocker/app:latest
                networks:
                        - appnet
                volumes:
                        - ./application:/var/www/html
                ports:
                        - 80:80
                working_dir: /var/www/html
```
and then restart:
```
docker-compose restart
```
and then exec in our app and list the current directory.
```
docker-compose exec app pwd
```
We notice that the current directory is still the root directory (`/`)

Docker exec (or `docker-compose exec`) runs on a currently-running Container.
Docker run (or `docker-composer run`) is going to create a new Container based on the docker-compose.yml:
```
docker-compose run app pwd
```

with docker-compose exec we can still do the following:
```
docker-compose exec app bash -c "cd /var/www/html && pwd"
```

If we destroy our Container and run it again, it will display the specified working directory:
```
docker-compose down

docker-compose up -d

docker-compose exec app pwd
```

---

## 06: Variables in Docker Compose
Source: https://serversforhackers.com/s/docker-in-dev-v2-ii

We will see how to use variables within a docker-compose.yml file

The syntax is: ${SOME_VAR_NAME}.

These specifically are environment variables.

---

We have APP_PORT AND DB_PORT. We can set those in two ways:

First, we can export the variables so they're available to sub-processes:
```
export APP_PORT=8080
export DB_PORT=33060

# Our `docker-compose.yml` file will use the above variables
docker-compose up -d
```

Second, we can set them inline as we run the docker-compose command:
```
APP_PORT=8080 DB_PORT=33060 docker-compose up -d
```

---

The third option is through the .env file.

When docker-compose runs it will look for an `.env` file in the root directory where the docker-compose.yml exists.
For laravel applications that come with an `.env` file by default this is no issue.

Additionally we can move the docker-compose.yml file inside the `application` directory, but
for the sake of this lesson we can create an additional `.env` file in our projects' root directory
which will not be shared with laravel and will be only for docker-compose:
```
vim .env
```
and add the following variable:
```
APP_PORT=8080
DB_PORT=33060
```
then we can update our docker-compose.yml file like so:
```
version: '3'
services:
        app:
                build:
                        context: ./docker/app
                        dockerfile: ./docker/app/Dockerfile
                image: shippingdocker/app:latest
                networks:
                        - appnet
                volumes:
                        - ./application:/var/www/html
                ports:
                        - ${APP_PORT}:80
```
and further down:
```
        db:
                image: mysql:5.7
                environment:
                        # The following will be used the first time the service is initialized.
                        #  for any subsequent updates to the values below, the database must be recreated.
                        MYSQL_ROOT_PASSWORD: root
                        MYSQL_DATABASE: homestead
                        MYSQL_USER: homestead
                        MYSQL_PASSWORD: secret
                ports:
                        - ${DB_PORT}:3306
                networks:
                        - appnet
                volumes:
                        - dbdata:/var/lib/mysql
```
and then:
```
docker-compose ps
```
then:
```
docker-compose down && docker-compose up -d
```

---

## 07: Adding a NodeJS Service
Source: https://serversforhackers.com/c/div-adding-a-build-service

We add a node service that we can use to build our static assets:

We update the `docker-compose.yml` file by adding the node service (after the `db` service):
```
  node:
    build:
      context: ./docker/node
      dockerfile: Dockerfile
    image: shippingdocker/node:latest
    networks:
      - appnet
    volumes:
      - .:/opt
    working_dir: /opt
    command: echo hi
```

Then we will create a new directory under `./docker` named `node` and add its `Dockerfile`:

We build this image also, so we can add things like git and yarn to it. This uses the official node base image.
Here's the Dockerfile:

```
FROM node:latest

LABEL maintainer="Dimitris Hantzis"

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb http://dl.yarnpkg.com/debian/ stable main" > /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y git yarn \
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```
Docker Compose takes care of building this image for us when we first spin up the environment.

Then will stop the currently running Containers (if any) and start them again to get the new node Container:
```
docker-compose down

docker-compose up -d
```
We can check the running Containers to see node running:
```
docker-compose ps
```

We can install yarn now in the `opt` working directory (as defined in the docker-compose.yml)

```
docker-compose run --rm node yarn install
```

Yarn will be installed (locally) inside the `./application/` directory (as defined in the `docker-compose.yml` under `volumes:`), creating the `yarn.lock` and `node_modules` directory

## 08: Dev Workflow Intro

We can build up a nice development workflow using a helper bash script.

This makes running commands within our Docker container easier. This will help us run commands like the following, easier:
```
docker-compose exec app bash -c "cd /var/www/html && php artisan migrate"
```

We also move our application files up a level so the Docker files are within the same directory.

```
docker-compose down

mv application/* ./

mv application/.* ./

rm -rf application
```

Because we moved the contents of the application directory, as well as deleting the directory,
we need to make some adjustments in the `docker-compose.yml` file for the shared directories:

The following:
```
...
    volumes:
      - ./application:/var/www/html
...
...
    volumes:
      - ./application:/opt
...
```

will become:
```
...
    volumes:
      - .:/var/www/html
...
...
    volumes:
      - .:/opt
...
```

We can check if anything is broken:
```
docker-compose ps

docker-compose up -d

curl localhost:80
```

Check the installation from your browser at `localhost:80` and check your database connection 
with your database UI.

We need to update laravels' .env file and add the following variables for our Ports:
```
APP_PORT=80
DB_PORT=33060
```

---

### develop Script

Create a new bash helper file, named `develop`:
```
touch ./application/develop
```
Add the following line in the `develop` helper:
```
echo #!/usr/bin/env bash
```
Make the file executable:
```
chmod +x develop
```

 ## 09: The Workflow
Source [https://serversforhackers.com/c/div-the-workflow](https://serversforhackers.com/c/div-the-workflow)

We will finish building our helper script to finalize our development workflow.

---

We can build a helper script - inside `./develop` - to accomplish the following:
- Pass any undefined commands to `docker-compose`
- Run `docker-compose ps` when we don't pass any arguments to the `develop` script
- Create a series of commands for `artisan`, `composer`, `yarn`, etc, setting the
    script to allow us to pass in any arguments to those commands.
    
All the commands we create allow us to run the corresponding commands within our running containers:

Edit the `./develop` file and add the following:
```
#!/usr/bin/env bash

if [ $# -gt 0 ]; then

    if [ "$1" == "start" ]; then
        docker-compose up -d

    elif [ "$1" == "stop" ]; then
        docker-compose down

    elif [ "$1" == "artisan" ] || [ "$1" == "art" ]; then
        shift 1
        docker-compose exec \
            app \
            php artisan "$@"

    elif [ "$1" == "composer" ] || [ "$1" == "comp" ]; then
        shift 1
        docker-compose exec \
            app \
            composer "$@"

    elif [ "$1" == "test" ]; then
        shift 1
        docker-compose exec \
            app \
            ./vendor/bin/phpunit "$@"

    elif [ "$1" == "npm" ]; then
        shift 1
        docker-compose run --rm \
            node \
            npm "$@"

    elif [ "$1" == "yarn" ]; then
        shift 1
        docker-compose run --rm \
            node \
            yarn "$@"

    elif [ "$1" == "gulp" ]; then
        shift 1
        docker-compose run --rm \
            node \
            ./node_modules/.bin/gulp "$@"
    else
        docker-compose "$@"
    fi

else
    docker-compose ps
fi
``` 

For the commands relating to the `node` Container we did not use `docker-compose exec` but `docker-composer run --rm`
This is because the node Container is not continuously running, so we use `run` to spin up a new container with the flag --rm to clean-up/destroy the container once it finishes running.

We can use the helper script like so:
```
# Run docker-compose ps
./develop ps

# Run docker-compose exec app bash
./develop exec app bash

# Run docker-compose exec app php artisan make:controller CustomController
./develop art make:controller SomeController

# Run docker-compose exec app composer require predis/predis
./develop composer require predis/predis

# Run docker-compose exec app ./vendor/bin/phpunit
./develop test

# Run commands against the node Container:
./develop yarn ...
./develop npm ...
./develop gulp ...
```

---

Walk-through of Course by: [Server For Hackers](https://serversforhackers.com)

- [dimitrioschantzis.com](http://www.dimitrioschantzis.com)
- <https://github.com/dchantzis>
