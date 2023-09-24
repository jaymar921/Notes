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

Build an image,

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
