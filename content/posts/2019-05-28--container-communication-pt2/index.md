---
title: Communication Between Containers (pt. 2)
subTitle: Making services which talk to each other
category: "containers"
cover: container.jpg
---

This post is the second part of a 3 article series
on making containers communicate with one another, especially
in the context of kubernetes. In this part of the article,
we'll be making our *Foo* and *Bar* services into containers,
launching them with docker-compose, and validating that they
can communicate with one another. Let's go ahead and get
started.

The source code for this article can be found in
[its github repo](https://github.com/briankopp/container-communication).

## Bar Service Dockerfile

Our first step will be to make a docker image of the
*Bar* service. We'll begin by adding a `Dockerfile`.

```dockerfile
# start with a small image
FROM node:10.15-alpine

# setup working directory
RUN mkdir -p /usr/app
WORKDIR /usr/app

# copy over files
COPY . .

# install dependencies
RUN npm install --production

# open up the port for node to listen on
EXPOSE 3000

# start the service
CMD npm run start
```

You'll want to also add a `.dockerignore` file to make
sure that your `node_modules`, `.env`, and other
files don't make it out onto your docker image.

```text
.gitignore
node_modules
.env
```

Next, we need to build the docker image. Run this command from the
`bar` folder.

```bash
docker build . -t your-organization/container-communication-bar:1.0.0
```

Take a look at the docker images to make sure things look right.

```bash
docker images
```

The image should be <100 MB.

Let's fire it up to make sure it works!

```bash
docker run --rm -p 127.0.0.1:3000:3000 your-organization/container-communication-bar:1.0.0
```

Here, the `-p` flag tells docker to listen on a public port and forward
the request into the container another port. In this case, we'll
have the docker container listen to localhost on port 3000 and send the
request into the container on port 3000. Exactly what we're expecting.

In another terminal, hit `curl localhost:3000`, and you should see the
familiar `Hello from BAR` message.

To stop the container, hit control+C from the terminal you started it in.
The `--rm` flag we used when we started the container cleans up the docker's
resources after it has shut down.

Next, let's write the Dockerfile for the foo service.

## Foo Service Dockerfile

Let's write the `Dockerfile` for the *Foo* service. This will be
largely the same as the *Bar* service `Dockerfile`.

```dockerfile
# start with a small image
FROM node:10.15-alpine

# setup working directory
RUN mkdir -p /usr/app
WORKDIR /usr/app
ENV NODE_ENV production

# copy over files
COPY . .

# install dependencies
RUN npm install --production

# open up the port for node to listen on
EXPOSE 3000

# start the service
CMD npm run start
```

I'm setting an environment variable here. Remember in the first article,
we were using the `dotenv` package to handle environment variables from a
.env file. But we only saved that as a development dependency. When
`npm install --production` is called, `dotenv` won't be installed. But it
was only invoked if the NODE_ENV environment variable wasn't set to `production`.
Hence, the Dockerfile assigns this environment variable.

I did not specify the BAR_URL environment variable or PORT environment variable.
Those may be specific to the deployment, so I want whoever is using this docker
image to be able to specify their own environment variables for these values.

Add a `.dockerignore` file as before, then run the following command
to build the docker image:
`docker build . -t your-organization/container-communication-foo:1.0.0`

## Docker Compose

Next, we need to get these two docker images up and running
and able to communicate with one another. For local testing,
we'll use `docker-compose`, which is great for this sort of thing.

Create a file called `docker-compose.yaml` in the directory
above your foo/ and bar/ directories.

```yaml
version: '3'
services:
  bar:
    image: "your-organization/container-communication-bar:1.0.0"
    ports:
    - "3000:3000"
  foo:
    image: "your-organization/container-communication-foo:1.0.0"
    ports:
    - "3001:3000"
    environment:
    - BAR_URL=http://bar:3000
```

The *bar* service will be exposed on port 3000, and the *foo*
service will be exposed on port 3001. In the *foo* section, we are
passing in the `BAR_URL` environment variable.
You'll notice the URL is not very verbose. When you run docker-compose,
it will set up a network bridge for the services to communicate with
one another at the hostname of the service name, in this case the hostname
is `bar`. Very handy!

Let's fire this sucker up!

```bash
docker-compose up
```

You should see that both services are up and running, each on port 3000.
docker-compose exposes separate ports on localhost, 3000 and 3001 for bar
and foo, respectively, but internally sends the requests to port
3000 for each container.

Try running curl to make sure the containers are communicating
with one another!

```bash
curl http://localhost:3000
# Hello from BAR
curl http://localhost:3001
# Hello from BAR from inside FOO
```

Use control+C to stop docker-compose.

## Push your Images

Make sure to push your images to a container registry like so:

```bash
docker push your-organization/container-communication-bar:1.0.0
docker push your-organization/container-communication-foo:1.0.0
```

## Conclusion

Well, that's it for part 2 of this series. Here we showed how to make
nice, small docker images for the *Foo* and *Bar* services. We
made a docker-compose file to run them next to one another in separate
containers, while maintaining the ability to communicate with one another.

This technique is very useful for running your development environment locally,
or for spinning up docker containers to run integration tests in your CI/CD
pipeline.

The last article in this series will be all about kubernetes. We'll set up
deployment manifests for these two apps, configure their services, and
make sure we specify the *bar* hostname correctly for inter-service
communication.

Thanks! And hope to see you in the final article!
