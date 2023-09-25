# Understanding dockerfiles

Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.

Inside the Dockerfile

```
FROM    node:alpine / ASP.NET Core / ...
LABEL author="Jayharron Mar Abejar"
ENV NODE_ENV=production
WORKDIR /var/www  <- directory of where you are working at, example root folder of the project
COPY . .  <- source, working dir. example: copy build/ to /workdir
RUN npm install  <- run a command, example run npm install
EXPOSE 3000 <- expose the port 3000, that will be what this container will be listening on
ENTRYPOINT ["node", "server.js"] <- what will be the first command that we're going to run to start this up.
```

Build an image

```
> docker built -t <name> .
```

-t stands for tag, short for --tag
. stands for build context, where is the Dockerfile? relative to where you are running this command

Build an image with registry

```
> docker build -t <registry>/<name>:tag .
```

example:

```
> docker build -t jaymar/photolib:1.0 .
```

### Docker Image Commands

List Docker Images

```
> docker images
```

Remove an image

```
> docker rmi <imageId>
```

### Bridge network

Creating a bridge network

```
docker network create

// another way
docker network create --driver bridge network_name
```

Showing list of network

```
docker network ls
```

Remove network

```
docker network rm [network]
```

### Running a Database container in a Network

```
docker run -d --net=network_name --name=mongodb mongo
                   ^                    ^
         Bridge network to use    Database container name
```

Note that `--net` or `--network` can be used

### Building and Running Multiple Containers with Docker Compose

Docker Compose Features

- Define services using a YAML configuration file
- Build one or more images
- Start and stop services
- View the status of running services
- Stream the log output of running services

Docker Compose Workflow

`Build Services` -> `Start Up Services` -> `Tear Down Services`

### Docker compose services

inside the `docker-compose.yml`, note that the indentation matters

```yml
version: "3.x"
services:
  app:
    container_name: myapp_1
    image: myapp_1
    build:
      context: . # (.) root folder
      dockerfile: /path/to/Dockerfile
    ports:
      - "3000:3000" # external_port : internal_port
    networks:
      - network_name
    depends_on:
      - mongodb # ensures that the mongodb runs first before this service, but it wont make sure that it's ready to accept connections

  mongodb:
    container_name: mongodb
    image: mongo
    networks:
      - network_name

networks:
  network_name:
    driver: bridge
```

### Docker Compose Commands

- `docker-compose build` - build all the apps specified in the services
- `docker-compose up` - run the containers/services
- `docker-compose down` - shutdown all containers/services and delete them
