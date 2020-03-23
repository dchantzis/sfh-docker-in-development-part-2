**Last Updated: 2020**

**Git Repository**: [https://github.com/dchantzis/sfh-docker-in-development-part-2](https://github.com/dchantzis/sfh-docker-in-development-part-2)

# Servers for Hackers: Docker in Development Part II
Source: https://serversforhackers.com/s/docker-in-dev-v2-ii

We continue the Docker in Dev course by using Docker Compose and building a development workflow. The first course can be found [here](https://github.com/dchantzis/sfh-docker-in-development-part-1)

## Table of Contents
1. [Docker Compose Intro](#01-docker-compose-intro)
2. [Docker Compose Services](#02-docker-compose-services)
3. [Compose and Volumes](#03-compose-and-volumes)
4. [Our App Service](#04-our-app-service)
5. [The Working Directory](#05-the-working-directory)
6. [Variables in Docker Compose](#06-variables-in-docker-compose)
7. [Adding a NodeJS Service](#07-adding-a-nodejs-service)
8. [Dev Workflow Intro](#08-dev-workflow-intro)
9. [The Workflow](#09-the-workflow)

---

## 01. Docker Compose Intro
Source: [https://serversforhackers.com/c/div-docker-compose-intro](https://serversforhackers.com/c/div-docker-compose-intro)

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

## 02. Docker Compose Services
Source: [https://serversforhackers.com/c/div-docker-compose-services](https://serversforhackers.com/c/div-docker-compose-services)

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

## 03. Compose and Volumes
Source: [https://serversforhackers.com/s/docker-in-dev-v2-ii](https://serversforhackers.com/s/docker-in-dev-v2-ii)

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

## 04. Our App Service
Source: [https://serversforhackers.com/c/div-our-app-service](https://serversforhackers.com/c/div-our-app-service)

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

## 05. The Working Directory
Source: [https://serversforhackers.com/c/div-working-directory](https://serversforhackers.com/c/div-working-directory)

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

## 06. Variables in Docker Compose
Source: [https://serversforhackers.com/s/docker-in-dev-v2-ii](https://serversforhackers.com/s/docker-in-dev-v2-ii)

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

## 07. Adding a NodeJS Service
Source: [https://serversforhackers.com/c/div-adding-a-build-service](https://serversforhackers.com/c/div-adding-a-build-service)

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

## 08. Dev Workflow Intro
Source: [https://serversforhackers.com/c/div-dev-workflow-intro](https://serversforhackers.com/c/div-dev-workflow-intro)

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

## 09. The Workflow
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
