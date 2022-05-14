## Setup Continuous Deployment For Your Django Project With Docker Compose, Caddy and GitHub

*There are many services that offer easy and continous deployments, for example Heroku or Dokku. But you might want to consider doing it yourself, Heroku is expensive and Dokku won't scale past one server, using Docker ensures that you will be able to deploy with more advanced and scalable concepts like Kubernetes when the time comes.*

## Prerequisites

- An Ubuntu VPS ([I recommend Hetzner](/vps-comparison))
- Basic understanding of Ubuntu + Terminal
- Your code in a GitHub repository
- PostgreSQL as the database for your Django Project (you will still be able to follow most of this guide if you use a different one)

## Why I wrote this tutorial

This tutorial builds on the official docs, which are great to get started, but don't include django-specific information and explain all the concepts, all in one place. That said, they are good references along the way.

* [Python Quickstart](https://docs.docker.com/language/python/)
* [Docker CI/CD Docs](https://docs.docker.com/ci-cd/github-actions/)
* [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
* [Compose File Reference](https://docs.docker.com/compose/compose-file/)
* [Compose Quickstart](https://docs.docker.com/compose/gettingstarted/)

## Install Docker and Docker Compose

Follow the [official Docker Installation docs](https://docs.docker.com/engine/install/ubuntu/). Docker Compose does not need to be installed as a seperate binary anymore, but can be installed from the docker apt repository following the installation of Docker: `sudo apt install docker-compose-plugin`. *Note that you will have to run `docker compose` instead of `docker-compose` to use it. For more information, please see the [official Docker Compose installation docs](https://docs.docker.com/compose/install/#install-compose-on-linux-systems).

I would also recommend creating a directory to store all your environment and compose files in.

You don't have to install Docker on your local development machine, unless you also want to use it for development.

Login with `docker login`, so that you will be able to access your private repositories.

## Create a Docker Hub account

This will be used later to push and pull updates of your application to/from. It comes included with one free private repository. Create it [here](https://hub.docker.com/) and then create a repository (make sure to set it to private) and remember the name, or use the name of your GitHub repository.

## The Dockerfile

If you are using VS Code, install the Docker extension to enable IntelliSense support -- especially helpful if you are new to Docker.

Start with the `FROM` command to specify an existing docker image to use as a starting point:

```Dockerfile
FROM python:3.9-alpine
```
In our case we use the alpine linux variant of the official python image. Alpine linux is used a lot for docker, because of it's small size.

Next we will use the `WORKDIR` command to set the working directory for all subsequent commands. This is recommended, as using the root folder could cause issues. You can name it whatever you want.

```Dockerfile
WORKDIR /app
```

Continuing, the `COPY` command copies files from the folder `docker build` (the command building the docker image) is running on, into the container.

```Dockerfile
COPY requirements.txt requirements.txt
```
We use the requirements.txt file to install all required python dependencies. Create it using `pip3 freeze > requirements.txt` or, if you are using `pipenv`, `pipenv run pip3 freeze > requirements.txt`

Before we can install the dependencies, we need to install support libraries for the postgres database driver django uses, psycopg2. Use the `RUN` command, which executes a command in the container. *This is not necessary if you are not using postgres, although other steps may be necessary.*

```Dockerfile
RUN apk update && apk add postgresql-dev python3-dev gcc musl-dev
```
If you didn't use alpine, this command will not work and you will have to execute the appropriate command using the package manager of your distro.

Finally, install the dependencies and copy your code:

```Dockerfile
RUN pip3 install -r requirements.txt
COPY . .
```

A few notes about this:
* Make sure to create a `.dockerignore` file in your repository and add your environment file(s) to it to prevent them from ending up in the image
* Why do we copy the requirements before copying the whole app? Docker caches every command/line in your Dockerfile as a layer, so it only needs to regenerate the layers that changed.

Use the `CMD` command to execute a script that should run when a container starts using this image.

```Dockerfile
RUN chmod +x gunicorn.sh
CMD [ "./gunicorn.sh" ]
```

If you were building an image for development, you would do:
```Dockerfile
CMD [ "python3", "manage.py", "runserver" ]
```

Before we can continue, add the script file `gunicorn.sh` to the root of your git repository and install `gunicorn` if it isn't already with `pip3 install gunicorn` (and rerun `pip freeze`):

```sh
#!/bin/sh
python3 manage.py migrate

gunicorn <your-django-project-name>.wsgi
```

You can add any other commands you want to run each time django (re)starts (i.e. your application is updated) before the gunicorn command.

Here's the complete Dockerfile:
```Dockerfile
FROM python:3.9-alpine

WORKDIR /app

COPY requirements.txt requirements.txt

RUN apk update && apk add postgresql-dev python3-dev gcc musl-dev

RUN pip3 install -r requirements.txt

COPY . .
CMD [ "./gunicorn.sh" ]
```

## Setup Continuous Deployment

Follow the [official docs for this step](https://docs.docker.com/language/python/configure-ci-cd/), as there is nothing to add here. Push to your GitHub repository, and the GitHub Action will build and push your image to Docker Hub, where your server will pull it from.

## Docker Compose

The final part of this guide. Your Docker Compose file will manage 4 services required to run your app.

* Your app itself
* Your Database (Postgres)
* A web server (Caddy in this guide, [which I like for it's automatic SSL/HTTPS and easy configuration](/caddy-reverse-proxy))
* Watchtower, which will automatically update your containers to the latest image, whenever you push it to Docker Hub.

The Docker Compose file is a YAML file that lets you organise the configuration for and run multiple docker containers.

You specify volumes, which will store persistent data (like static files or your database) and the configuration for each service. The services are accessible by the other ones using their names as a hostname. To make a service visible to the host network (and the internet if the port isn't blocked by your firewall), we specify which ports we want published. This allows to isolate all networking from the host and the internet and only expose the web server.

Let's configure each service individually, starting with your brand new docker image (your app). Start your `docker-compose.yml` file with:

```yml
version: '3'

services:
  app:
    image: index.docker.io/<docker-hub-username>/<docker-hub-repository>
    restart: always
    environment:
      GUNICORN_CMD_ARGS: "--bind=0.0.0.0:8000"
    env_file:
      - .app.env
    depends_on:
      - db
    volumes:
      - app-static:/var/www/django/static

volumes:
 db-data: {}
 caddy-data: {}
 caddy-config: {}
 app-static: {}
```
This defines that our file is using the docker compose version 3 format, and lists all volumes we will need for our case.

* `app:` defines the app service, it will be visible to other services under the `app` hostname.
* `image:` tells docker compose which image to use. You need to prefix `index.docker.io/`, so Watchtower will be able to find it.
* `environment:` configures environment variables, the environment variable is necessary to have django/gunicorn be accessible from the hostname `app`.
* `env_file:` lets you list files to load more environment variables from, you can keep API keys, etc. in here. If you use a custom `settings.py` files for configuration, add them under volumes using the `<host-path>:<container-path>` pattern.
* `depends_on:` specifies other services this one depends_on. They determine the order in which docker compose starts them. `db` is added, as Django won't run without the database.
* `volumes:` lets you add docker volumes to this service. Your app will probably only require one, to save static files to. Make sure to set `STATIC_ROOT` to the path specified here in your `settings.py`
* `restart: always` tells docker compose to always restart your app should it crash.

Create a file `.app.env` on your server with all the configuration required for your project. If you use a different method to store environment specific variables, you can skip this and remove it from the docker compose file.

Let's continue with the database. Add the following snippet to your services:
```yml
db:
    image: postgres:alpine
    restart: always
    env_file:
      - .db.env
    volumes:
      - db-data:/var/lib/postgresql/data
```

This one is quite simple, we use a different `.env` file for the database configuration and mount a volume to make persist your data across restarts. Again we use the alpine image to reduce size.

Create a new file `.db.env` with your database configuration:
```env
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
```

If `POSTGRES_DB` is not set, it will use the username. For more configuration options, [visit the official docs](https://hub.docker.com/_/postgres)

Next we will set up the web server, which will be the only service exposed to our host network:

```yml
proxy:
    image: caddy:alpine
    restart: always
    volumes:
      - caddy-data:/data
      - caddy-config:/config
      - ./Caddyfile:/etc/caddy/Caddyfile
      - app-static:/var/www/django/static:ro
    ports:
      - 80:80
      - 443:443
    depends_on:
      - db
      - app
```

Again, not much new here. 
* Use `ports:` to specify the ports that will be visible to the host system (you could even change the port number)
* Use `depends_on:` to make sure that the web server is the last one to start.
* Mount all required volumes, and your Caddyfile, which is the file in which you will save your configuration. This includes your static files exported by Django with the `:ro` suffix, specifying that caddy only has read-only access to these files.

Next configure it:
```Caddyfile
<your-domain>

handle_path /static {
        root * /var/www/django
        file_server
}

handle {
        reverse_proxy app:8000
}
```
This is almost all you need for a web server with automatic HTTPS by Let's Encrypt. If your needs are more advanced, [review the documentation](https://caddyserver.com/).

A few things to note though:
* If your `STATIC_URL` is not `/static` you will change it here, same with `/var/www/django`, which is also specified in the volumes of this and the app service.
* Use `app` as the hostname as this is the name you set for that service.

In your `settings.py`:
* Make sure that `SECURE_SSL_REDIRECT` is `False` as this is handled by Caddy automatically
* Set `SECURE_PROXY_SSL_HEADER` to `('HTTP_X_FORWARDED_PROTO', 'https')` to prevent an infinite redirect loop

Finally add watchtower to automatically update your app:
```yml
watchtower:
    restart: always
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/config.json
    command: --interval 30
```

This exposes docker to watchtower (required for it to work) and sets the frequency with which it checks for updates to 30 seconds. More configuration options can be found [here](https://containrrr.dev/watchtower/usage-overview/).

With this you are done. Here's the final file for reference:
```yml
version: '3'

services:
  app:
    image: index.docker.io/<docker-hub-username>/<docker-hub-repository>
    restart: always
    environment:
      GUNICORN_CMD_ARGS: "--bind=0.0.0.0:8000"
    env_file:
      - .app.env
    depends_on:
      - db
    volumes:
      - app-static:/var/www/django/static
  db:
    image: postgres:alpine
    restart: always
    env_file:
      - .db.env
    volumes:
      - db-data:/var/lib/postgresql/data
  proxy:
    image: caddy:alpine
    restart: always
    volumes:
      - caddy-data:/data
      - caddy-config:/config
      - ./Caddyfile:/etc/caddy/Caddyfile
      - app-static:/var/www/django/static:ro
    ports:
      - 80:80
      - 443:443
    depends_on:
      - db
      - app
  watchtower:
    restart: always
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/config.json
    command: --interval 30

volumes:
 db-data: {}
 caddy-data: {}
 caddy-config: {}
 app-static: {}
```

## Manage your deployment
To bring this to an conclusion, here are the most important commands to manage your docker compose deployment, note that all of these need to be run with `sudo`:

* `docker compose up` starts all your services and streams the logs to your terminal (stdout), exit with `CTRL+C`
* `docker compose up -d` starts all your services in detached mode (the command exits when all are services are started, you will likely want to use this command)
* `docker compose down` shuts down all services
* `docker compose logs <service>` shows all logs for a specific service, leave it out to show the logs of all services
* `docker compose pull`: lets you manually pull the newest version of your images