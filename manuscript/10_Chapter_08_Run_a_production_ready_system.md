{blockquote}
Vilos Cohaagen said troops would be used to ensure full production.

Total Recall, 1990
{/blockquote}
Now that we developed a repository that connects with PostgreSQL we can discuss how to properly set up the application to run a production-ready system. This part is not strictly related to the clean architecture, but I think it's worth completing the example, showing how the system that we designed can end up being the core of a real web application.

Clearly, the definition "production-ready" refers to many different configuration that ultimately depend on the load and the business requirements of the system. As the goal is to show a complete example and not to cover real production requirements I will show a solution that uses real external systems like PostgreSQL and Nginx, without being too concerned about performances.

## Build a web stack

Now that we successfully containerised the tests we might try to devise a production-ready setup of the whole application, running both a web server and a database in Docker containers. Once again, I will follow the approach that I show in the series of posts I mentioned in one of the previous sections.

To run a production-ready infrastructure we need to put a WSGI server in front of the web framework and a Web server in front of it. We will also need to run a database container that we will initialise only once.

The steps towards a production-ready configuration are not complicated and the final setup won't be ultimately too different form what we already did for the tests. We need to


* Create a JSON configuration with environment variables suitable for production
* Create a suitable configuration for Docker Compose and configure the containers
* Add commands to `manage.py` that allow us to control the processes

Let's create the file `config/production.json`, which is very similar to the one we created for the tests

{caption: "`config/production.json`"}
``` json
[
  {
    "name": "FLASK_ENV",
    "value": "production"
  },
  {
    "name": "FLASK_CONFIG",
    "value": "production"
  },
  {
    "name": "POSTGRES_DB",
    "value": "postgres"
  },
  {
    "name": "POSTGRES_USER",
    "value": "postgres"
  },
  {
    "name": "POSTGRES_HOSTNAME",
    "value": "localhost"
  },
  {
    "name": "POSTGRES_PORT",
    "value": "5432"
  },
  {
    "name": "POSTGRES_PASSWORD",
    "value": "postgres"
  },
  {
    "name": "APPLICATION_DB",
    "value": "application"
  }
]
```
Please note that now both `FLASK_ENV` and `FLASK_CONFIG` are set to `production`. Please remember that the first is an internal Flask variable with two possible fixed values (`development` and `production`), while the second one is an arbitrary name that has the final effect of loading a specific configuration object (`ProductionConfig` in this case). I also changed `POSTGRES_PORT` back to the default `5432` and `APPLICATION_DB` to `application` (an arbitrary name).

Let's define which containers we want to run in our production environment, and how we want to connect them. We need a production-ready database and I will use Postgres, as I already did during the tests. Then we need to wrap Flask with a production HTTP server, and for this job I will use `gunicorn`. Last, we need a Web Server to act as load balancer.

The file `docker/production.yml` will contain the Docker Compose configuration, according to the convention we defined in `manage.py`

{caption: "`docker/production.yml`"}
``` yaml
version: '3.8'

services:
  db:
    image: postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
  web:
    build:
      context: ${PWD}
      dockerfile: docker/web/Dockerfile.production
    environment:
      FLASK_ENV: ${FLASK_ENV}
      FLASK_CONFIG: ${FLASK_CONFIG}
      APPLICATION_DB: ${APPLICATION_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_HOSTNAME: "db"
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_PORT: ${POSTGRES_PORT}
    command: gunicorn -w 4 -b 0.0.0.0 wsgi:app
    volumes:
      - ${PWD}:/opt/code
  nginx:
    image: nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - 8080:8080

volumes:
  pgdata:
```
As you can see the Postgres configuration is not different from the one we used in the file `testing.yml`, but I added the option `volumes` (both in `db` and at the end of the file) that allows me to create a stable volume. If you don't do it, the database will be destroyed once you shut down the container.

The container `web` runs the Flask application through `gunicorn`. The environment variables come once again from the JSON configuration, and we need to define them because the application needs to know how to connect with the database and how to run the web framework. The command `gunicorn -w 4 -b 0.0.0.0 wsgi:app` loads the WSGI application we created in `wsgi.py` and runs it in 4 concurrent processes. This container is created using `docker/web/Dockerfile.production` which I still have to define.

The last container is `nginx`, which we will use as it is directly from the Docker Hub. The container runs Nginx with the configuration stored in `/etc/nginx/nginx.conf`, which is the file we overwrite with the local one `./nginx/nginx.conf`. Please note that I configured it to use port 8080 instead of the standard port 80 for HTTP to avoid clashing with other software that you might be running on your computer.

The Dockerfile for the web application is the following

{caption: "`docker/web/Dockerfile.production`"}
``` docker
FROM python:3

ENV PYTHONUNBUFFERED 1

RUN mkdir /opt/code
RUN mkdir /opt/requirements
WORKDIR /opt/code

ADD requirements /opt/requirements
RUN pip install -r /opt/requirements/prod.txt
```
This is a very simple container that uses the standard `python:3` image, where I added the production requirements contained in `requirements/prod.txt`. To make the Docker container work we need to add `gunicorn` to this last file

{caption: "`requirements/prod.txt`"}
``` text
Flask
SQLAlchemy
psycopg2
pymongo
gunicorn
```
The configuration for Nginx is

{caption: "`docker/nginx/nginx.conf`"}
``` nginx
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    upstream app {
        server web:8000;
    }

    server {
        listen 8080;

        location / {
            proxy_pass         http://app;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
        }
    }
}
```
As for the rest of the project, this configuration is very basic and lacks some important parts that are mandatory in a real production environment, such as HTTPS. In its essence, though, it is however not too different from the configuration of a production-ready Nginx container.

As we will use Docker Compose, the script `manage.py` needs a simple change, which is a command that wraps `docker-compose` itself. We need the script to just initialise environment variables according to the content of the JSON configuration file and then run Docker Compose. As we already have the function `docker_compose_cmdline` the job is pretty simple

{caption: "`manage.py`"}
``` python
# Ensure an environment variable exists and has a value
import os
import json
import signal
import subprocess
import time

...

def setenv(variable, default):
    os.environ[variable] = os.getenv(variable, default)


setenv("APPLICATION_CONFIG", "production")

APPLICATION_CONFIG_PATH = "config"
DOCKER_PATH = "docker"

...

@cli.command(context_settings={"ignore_unknown_options": True})
@click.argument("subcommand", nargs=-1, type=click.Path())
def compose(subcommand):
    configure_app(os.getenv("APPLICATION_CONFIG"))
    cmdline = docker_compose_cmdline() + list(subcommand)

    try:
        p = subprocess.Popen(cmdline)
        p.wait()
    except KeyboardInterrupt:
        p.send_signal(signal.SIGINT)
        p.wait()
```
As you can see I forced the variable `APPLICATION_CONFIG` to be `production` if not specified. Usually, my default configuration is the development one, but in this simple case I haven't defined one, so this will do for now.

The new command is `compose`, that leverages Click's `argument` decorator to collect subcommands and attach them to the Docker Compose command line. I also use the `signal` library, which I added to the imports, to control keyboard interruptions.

{blurb, class: tip}
<https://github.com/pycabook/rentomatic/tree/ed2-c08-s01>
{/blurb}
When all this changes are in place we can test the application Dockerfile building the container.

``` sh
$ ./manage.py compose build web
```
This command runs the Click command `compose` that first reads environment variables from the file `config/production.json`, and then runs `docker-compose` passing it the subcommand `build web`.

You output should be the following (with different image IDs)

``` text
Building web
Step 1/7 : FROM python:3
 ---> 768307cdb962
Step 2/7 : ENV PYTHONUNBUFFERED 1
 ---> Using cache
 ---> 0f2bb60286d3
Step 3/7 : RUN mkdir /opt/code
 ---> Using cache
 ---> e1278ef74291
Step 4/7 : RUN mkdir /opt/requirements
 ---> Using cache
 ---> 6d23f8abf0eb
Step 5/7 : WORKDIR /opt/code
 ---> Using cache
 ---> 8a3b6ae6d21c
Step 6/7 : ADD requirements /opt/requirements
 ---> Using cache
 ---> 75133f765531
Step 7/7 : RUN pip install -r /opt/requirements/prod.txt
 ---> Using cache
 ---> db644df9ba04

Successfully built db644df9ba04
Successfully tagged production_web:latest
```
If this is successful you can run Docker Compose

``` text
$ ./manage.py compose up -d
Creating production_web_1   ... done
Creating production_db_1    ... done
Creating production_nginx_1 ... done
```
and the output of `docker ps` should show three containers running

``` text
$ docker ps
... IMAGE          ...   PORTS                            NAMES
... nginx          ...   80/tcp, 0.0.0.0:8080->8080/tcp   production_nginx_1
... postgres       ...   0.0.0.0:5432->5432/tcp           production_db_1
... production_web ...                                    production_web_1
```
Note that I removed several columns to make the output readable.

At this point we can open <http://localhost:8080/rooms> with our browser and see the result of the HTTP request received by Nginx, passed to gunicorn, and processed by Flask using the use case `room_list_use_case`.

The application is not actually using the database yet, as the Flask endpoint `room_list` in `application/rest/room.py` initialises the class `MemRepo` and loads it with some static values, which are the ones we see in our browser.

## Connect to a production-ready database

Thanks to the common interface between repositories moving from the memory-based `MemRepo` to `PostgresRepo` is very simple. Clearly, the external database will not contain any data initially, so the response of the use case will be empty.

First of all, let's move the application to the Postgres repository. The new version of the endpoint is

{caption: "`application/rest/room.py`"}
``` python
import os
import json

from flask import Blueprint, request, Response

from rentomatic.repository.postgresrepo import PostgresRepo
from rentomatic.use_cases.room_list import room_list_use_case
from rentomatic.serializers.room import RoomJsonEncoder
from rentomatic.requests.room_list import build_room_list_request
from rentomatic.responses import ResponseTypes

blueprint = Blueprint("room", __name__)

STATUS_CODES = {
    ResponseTypes.SUCCESS: 200,
    ResponseTypes.RESOURCE_ERROR: 404,
    ResponseTypes.PARAMETERS_ERROR: 400,
    ResponseTypes.SYSTEM_ERROR: 500,
}

postgres_configuration = {
    "POSTGRES_USER": os.environ["POSTGRES_USER"],
    "POSTGRES_PASSWORD": os.environ["POSTGRES_PASSWORD"],
    "POSTGRES_HOSTNAME": os.environ["POSTGRES_HOSTNAME"],
    "POSTGRES_PORT": os.environ["POSTGRES_PORT"],
    "APPLICATION_DB": os.environ["APPLICATION_DB"],
}


@blueprint.route("/rooms", methods=["GET"])
def room_list():
    qrystr_params = {
        "filters": {},
    }

    for arg, values in request.args.items():
        if arg.startswith("filter_"):
            qrystr_params["filters"][arg.replace("filter_", "")] = values

    request_object = build_room_list_request(
        filters=qrystr_params["filters"]
    )

    repo = PostgresRepo(postgres_configuration)
    response = room_list_use_case(repo, request_object)

    return Response(
        json.dumps(response.value, cls=RoomJsonEncoder),
        mimetype="application/json",
        status=STATUS_CODES[response.type],
    )
```
As you can see the main change is that `repo = MemRepo(rooms)` becomes `repo = PostgresRepo(postgres_configuration)`. Such a simple change is made possible by the clean architecture and its strict layered approach. The only other notable change is that we replaced the initial data for the memory-based repository with a dictionary containing connection data, which comes from the environment variables set by the management script.

This is enough to make the application connect to the Postgres database that we are running in a container, but as I mentioned we also need to initialise the database. The bare minimum that we need is an empty database with the correct name. Remember that in this particular setup we use for the application a different database (`APPLICATION_DB`) from the one that the Postgres container creates automatically at startup (`POSTGRES_DB`). I added a specific command to the management script to perform this task

{caption: "`manage.py`"}
``` python
@cli.command()
def init_postgres():
    configure_app(os.getenv("APPLICATION_CONFIG"))

    try:
        run_sql([f"CREATE DATABASE {os.getenv('APPLICATION_DB')}"])
    except psycopg2.errors.DuplicateDatabase:
        print(
            (
                f"The database {os.getenv('APPLICATION_DB')} already",
                "exists and will not be recreated",
            )
        )
```
Now spin up your containers with `./manage.py compose up -d` and run this new command `./manage.py init-postgres` (mind the change between the name of the function `init_postgres` and the name of the command `init-postgres`). You only need to run this command once, but repeated executions will not affect the database.

If you open <http://localhost:8080/rooms> with your browser you will see a successful response, but no data, as the database is correctly connected but empty.

To see some data we need to write something into the database. This is normally done through specific endpoints, but for the sake of simplicity in this case we can just log into the database and add data manually. First, connect to the database with `psql`

``` sh
$ ./manage.py compose exec db psql -U postgres
```
Please note that we need to specify the user `postgres` that has been created passing that value as `POSTGRES_USER` to the container. We can use the command `\l` to see the available databases

``` text
psql (13.0 (Debian 13.0-1.pgdg100+1))
Type "help" for help.

postgres=# \l
                                  List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-------------+----------+----------+------------+------------+----------------------
 application | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
             |          |          |            |            | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
             |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# 
```
and the command `\c` to connect to the database `application`, which is the one we want to write to

``` text
postgres=# \c application 
You are now connected to database "application" as user "postgres".
application=# 
```
Now we can list the available tables with `\dt` and describe a table with `\d`

``` text
application=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | room | table | postgres
(1 row)

application=# \d room
                                     Table "public.room"
  Column   |         Type          | Nullable |             Default
-----------+-----------------------+----------+----------------------------------
 id        | integer               | not null | nextval('room_id_seq'::regclass)
 code      | character varying(36) | not null | 
 size      | integer               |          | 
 price     | integer               |          | 
 longitude | double precision      |          | 
 latitude  | double precision      |          | 
Indexes:
    "room_pkey" PRIMARY KEY, btree (id)

application=# 
```
Please note that I deleted the column `collation` to make the output more compact. We can now insert data using a simple SQL statement

``` sql
application=# INSERT INTO room(code, size, price, longitude, latitude) VALUES ('f853578c-fc0f-4e65-81b8-566c5dffa35a', 215, 39, -0.09998975, 51.75436293);
INSERT 0 1
```
You can verify that the table contains the new room with a `SELECT`

``` sql
application=# SELECT * FROM room;
 id |                 code                 | size | price |  longitude  |  latitude
----+--------------------------------------+------+-------+-------------+-----------
  1 | f853578c-fc0f-4e65-81b8-566c5dffa35a |  215 |    39 | -0.09998975 | 51.75436293
(1 row)
```
and open or refresh <http://localhost:8080/rooms> with the browser to see the value returned by our use case.

{blurb, class: tip}
<https://github.com/pycabook/rentomatic/tree/ed2-c08-s02>
{/blurb}
## Conclusions

This chapter concludes the overview of the clean architecture example. Starting from scratch, we created domain models, serializers, use cases, an in-memory storage system, a command-line interface and an HTTP endpoint. We then improved the whole system with a very generic request/response management code, that provides robust support for errors. Last, we implemented two new storage systems, using both a relational and a NoSQL database.

This is by no means a little achievement. Our architecture covers a very small use case, but is robust and fully tested. Whatever error we might find in the way we dealt with data, databases, requests, and so on, can be isolated and tamed much faster than in a system which doesn't have tests. Moreover, the decoupling philosophy not only allows us to provide support for multiple storage systems, but also to quickly implement new access protocols, or new serialisations for our objects.

