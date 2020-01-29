# Postgraphile as Library

The following guide describes how to run a network of Docker containers on a local machine, including one container for a PostgreSQL database and one container running PostGraphile.

It is the same as the repository **[Docker-PostgreSQL-PostGraphile](https://github.com/alexisrolland/docker-postgresql-postgraphile)** with one difference, the PostGraphile container runs a **Node.js application using PostGraphile as library** instead of running PostGraphile via CLI. Running PostGraphile as a library allows greater customization possibilities

Follow the steps provided in the repository [Docker-PostgreSQL-PostGraphile](https://github.com/alexisrolland/docker-postgresql-postgraphile) and come back to this guide to create the GraphQL container.

# Table of Contents

-   [Create PostGraphile Container](#create-postgraphile-container)
    -   [Update Environment Variables](#update-environment-variables)
    -   [Create Node.js Application](#create-nodejs-application)
    -   [Create PostGraphile Dockerfile](#create-postgraphile-dockerfile)
    -   [Update Docker Compose File](#update-docker-compose-file)
-   [Build Images And Run Containers](#build-images-and-run-containers)
    -   [Build Images](#build-images)
    -   [Run Containers](#run-containers)
    -   [Re-initialize The Database](#re-initialize-the-database)

# Create PostGraphile Container

At this stage, the repository should look like this.

```
/
├─ db/
|  ├─ init/
|  |  ├─ 00-database.sql
|  |  └─ 01-data.sql
|  └─ Dockerfile
├─ .env
└─ docker-compose.yml
```

## Update Environment Variables

Update the file `.env` to add the `PORT` and `DATABASE_URL` which will be used by PostGraphile to connect to the PostgreSQL database. Note the `DATABASE_URL` follows the syntax `postgres://<user>:<password>@db:5432/<db_name>`.

```
[...]
# GRAPHQL
# Parameters used by graphql container
DATABASE_URL=postgres://postgres:change_me@db:5432/forum_example
PORT=5433
```

## Create Node.js Application

Create a new folder `graphql` at the root of the repository. It will be used to store the files necessary to create the PostGraphile container. In the `graphql` folder, create a subfolder `src` and add a file `package.json` into it with the following content.

```json
{
    "name": "postgraphile-as-library",
    "version": "0.0.1",
    "description": "PostGraphile as a library in a dockerized Node.js application.",
    "author": "Alexis ROLLAND",
    "license": "Apache-2.0",
    "main": "server.js",
    "keywords": ["nodejs", "postgraphile"],
    "dependencies": {
        "postgraphile": "^4.5.5",
        "postgraphile-plugin-connection-filter": "^1.1.3"
    }
}
```

This file will be used by NPM package manager to install the dependencies in the Node.js container. In particular `postgraphile` and the excellent plugin `postgraphile-plugin-connection-filter`.

In the same `src` folder, create a new file `server.js` with the following content.

```js
const http = require("http");
const { postgraphile } = require("postgraphile");

http.createServer(
    postgraphile(process.env.DATABASE_URL, "public", {
        watchPg: true,
        graphiql: true,
        enhanceGraphiql: true
    })
).listen(process.env.PORT);
```

## Create PostGraphile Dockerfile

Create a new file `Dockerfile` in the `graphql` folder (not in the folder `src`) with the following content.

```dockerfile
FROM node:alpine
LABEL description="Instant high-performance GraphQL API for your PostgreSQL database https://github.com/graphile/postgraphile"

# Set Node.js app folder
RUN mkdir -p /home/node/app/node_modules && chown -R node:node /home/node/app
WORKDIR /home/node/app

# Copy dependencies
COPY ./src/package*.json .

# Install dependencies
USER node
RUN npm install

# Copy application files
COPY --chown=node:node ./src .

EXPOSE 8080
CMD [ "node", "server.js" ]
```

## Update Docker Compose File

Update the file `docker-compose.yml` under the `services` section to include the GraphQL service.

```yml
version: "3.3"
services:
    db: [...]

    graphql:
        container_name: forum-example-graphql
        restart: always
        image: forum-example-graphql
        build:
            context: ./graphql
        env_file:
            - ./.env
        depends_on:
            - db
        networks:
            - network
        ports:
            - 5433:5433
[...]
```

At this stage, the repository should look like this.

```
/
├─ db/
|  ├─ init/
|  |  ├─ 00-database.sql
|  |  └─ 01-data.sql
|  └─ Dockerfile
├─ graphql/
|  ├─ src/
|  |  ├─ package.json
|  |  └─ server.js
|  └─ Dockerfile
├─ .env
└─ docker-compose.yml
```

# Build Images And Run Containers

## Build Images

You can build the Docker images by executing the following command from the root of the repository.

```
# Build all images in docker compose
$ docker-compose build

# Build database image only
$ docker-compose build db

# Build graphql image only
$ docker-compose build graphql
```

## Run Containers

You can run the Docker containers by executing the following command from the root of the repository. Note when running the database container for the first time, Docker will automatically create a Docker Volume to persist the data from the database. The Docker Volume is automatically named as `<your_repository_name>_db`.

```
# Run all containers
$ docker-compose up

# Run all containers as daemon (in background)
$ docker-compose up -d

# Run database container as daemon
$ docker-compose up -d db

# Run graphql container as daemon
$ docker-compose up -d graphql
```

Each container can be accessed at the following addresses. Note if you run Docker Toolbox on Windows Home, you can get your Docker machine IP address with the command `$ docker-machine ip default`.

<table>
    <tr>
        <th>Container</th>
        <th>Docker on Linux / Windows Pro</th>
        <th>Docker on Windows Home</th>
    </tr>
    <tr>
        <td>GraphQL API Documentation</td>
        <td>https://localhost:5433/graphiql</td>
        <td>https://your_docker_machine_ip:5433/graphiql</td>
    </tr>
    <tr>
        <td>GraphQL API</td>
        <td>https://localhost:5433/graphql</td>
        <td>https://your_docker_machine_ip:5433/graphql</td>
    </tr>
    <tr>
        <td>PostgreSQL Database</td>
        <td>host: localhost, port: 5432</td>
        <td>host: your_docker_machine_ip, port: 5432</td>
    </tr>
</table>

## Re-initialize The Database

In case you do changes to the database schema by modifying the files in `/db/init`, you will need to re-initialize the database to see these changes. This means you need to delete the Docker Volume, the database Docker Image and rebuild it.

```shell
# Stop running containers
$ docker-compose down

# List Docker volumes
$ docker volume ls

# Delete volume
$ docker volume rm <your_repository_name>_db

# Delete database image to force rebuild
$ docker rmi db

# Run containers (will automatically rebuild the image)
$ docker-compose up
```
