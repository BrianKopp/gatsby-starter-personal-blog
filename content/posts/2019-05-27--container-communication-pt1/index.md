---
title: Communication Between Containers (pt. 1)
subTitle: Making services which talk to each other
category: "containers"
cover: container.jpg
---

Containers are a great tool and solve a lot of problems.
Make a small set of code do one thing. Make it fast. Make it
resilient. Make it isolated. Make it readable. Make it
beautiful. The move to containers and microservices solves a
lot of problems, but it also adds complexity. One piece of complexity
is inter-service communication. You make all these different containers
that do one thing, but in order to do what the customer ultimately
wants, often we have to touch multiple services. How in the
heck do these different things speak to one another?
The answer to that question largely depends on the particular
implementation. However, this set of three articles covers a basic
example of inter-service communication using docker, docker-compose,
and kubernetes.

All the code for this tutorial can be found on my
[github repo](https://github.com/briankopp/container-communication).

## Project Setup

We'll make two minimal node express services. *Foo service* will
call *Bar service*. Let's get started.

```bash
mkdir container-communication
cd container-communication
git init
```

Next, we'll make the two node services, starting with the *Bar service*.

## Bar Service

```bash
mkdir bar
cd bar
npm init
```

Fill out the `npm init` prompts, and then modify your `package.json`
script object by adding a `start` command.

```json
{
  "name": "bar",
  "version": "1.0.0",
  "description": "node express app which replies 'Hello from BAR'",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "",
  "dependencies": {}
}
```

Next, we need to install our dependencies. The *Bar* service's
only job is to respond to an http request. It will just require
express.

```bash
npm install --save express
```

Now our *Bar* service's project structure looks like:

```text
bar
| - node_modules/
| - package.json
| - package-lock.json
```

Let's add a `.gitignore` file to keep the node_modules folder
out of source control.

```text
node_modules
build
npm-debug.log
.env
.DS_Store
```

Next, we need to implement the *Bar* service. Let's make an
`app.js` file.

```js
const express = require('express');

const app = express();
app.get('/', (barRequest, barResponse) => {
    barResponse.send('Hello from BAR');
});

module.exports = app;
```

This service really is as simple as possible. We require express,
and respond to the path `/` with `Hello from BAR`.

In our `package.json` file, our start command hits `index.js`, so
let's make an index file.

```js
const app = require('./app');

app.listen(3000, () => console.log('express "bar" app listening on port 3000'));
```

It's common to split the express app definition into its own file, e.g.
`app.js`, and have it started in an index file, e.g. `index.js`, in line
with the single-responsibility principle. `app.js` is responsible for
defining the app, and `index.js` is repsonsible for running it. This has its
advantages especially when you want to unit test `app.js`, but don't want the
`app.listen()` method invoked.

That should do it for the Bar service's implementation. Your project
directory should look like:

```text
bar
| - node_modules/
| - app.js
| - index.js
| - package.json
| - package-lock.json
```

Let's give it a test.
Open up two terminals. In one, run the command `npm run start` from
within the `boo` folder. You should get an output saying

```text
express "bar" app listening on port 3000
```

In the other terminal, run the command `curl http://localhost:3000`.
You should get the response

```text
Hello from BAR
```

Great! Let's move on.

## Foo Service

The *Foo* service is pretty simple as well, except that it will
call the *Bar* service. Let's begin by making a folder called
`foo` next to the `bar` folder.

```bash
mkdir foo
cd foo
npm init
```

As before, fill out the npm init prompts, and then update your
`package.json` file's scripts object, like so.

```json
{
  "name": "foo",
  "version": "1.0.0",
  "description": "node express app which calls another api",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "",
  "dependencies": {}
}
```

Once again we'll need to install express. The *Foo* service
will need to call the *Bar* service. I'll also install the
*request* library to make that easier.

```bash
npm install --save express
npm install --save request
```

Add a `.gitignore` file like you did with the *Bar* service.

Next, let's set up the express app. Make a new file called
`app.js` with the following:

```js
const request = require('request');
const express = require('express');

const app = express();
app.get('/', (fooRequest, fooResponse) => {
    request.get(process.env.BAR_URL, { json: true }, (err, barResponse, barBody) => {
        if (err) {
            console.log(err);
            fooResponse.status(500);
            fooResponse.send('error in BAR request');
        } else {
            console.log('successfully received BAR response');
            fooResponse.status(barResponse.statusCode);
            fooResponse.send(barBody + ' from inside FOO');
        }
    });
});

module.exports = app;
```

There is a little more to this express app. Once again, only the root path
has a handler. In that handler, we call `request.get()`, which takes
a URL which we are getting from the `BAR_URL` environment variable
(we'll set that later), passes an option `{ json: true }`, and then a
callback function. In the callback, we check for an error. If an error
was present, we log it and then pass back a 500 - Internal Error response.
If no error was present, then we passthrough the *Bar* service's HTTP status
code as well as its body, along with an extra bit of text to prove it came
out of the *Foo* service.

Once again we'll want an `index.js` file.

```js
const app = require('./app');

app.listen(3000, () => console.log('express app listening on port 3000'));
```

This is the same `index.js` file as before.

You might notice a problem. We can't have two services both listening on
the same port when we're running on our own local machine. If
you try, you'll get an error. Let's modify our *Foo* service's port
by having an environment variable tell us which port to use.

```js
const app = require('./app');

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('express "foo" app listening on port ' + port));
```

The app will now run on whichever port we tell it to, or 3000 if none
is specified. For local testing, we'll want to give node the `PORT`
environment variable, as well as the `BAR_URL` environment variable.
An easy way to do that for testing is to add a `.env` file like so:

```text
BAR_URL=http://localhost:3000
PORT=3001
```

You won't see this .env file in my github repo because it's bad practice
to save .env files. Often sensitive information is stored in .env files.

Next, node needs to be told about this file, so we'll add a package
called `dotenv` which grabs variables and values from `.env` files
and hands them off to node. We won't be using this in production,
so we'll run the command `npm install --save-dev dotenv`.

We'll update our `index.js` file to the following:

```js
const app = require('./app');

if (process.env.NODE_ENV !== 'production') {
    require('dotenv').config();
}

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('express app listening on port ' + port));
```

Now, when we execute `npm run start`, you should get the output
`express "foo" app listening on port 3001`.

## Communicating Between Services

Close the other npm processes you started earlier using the control+C
command.

You'll need three terminals to do the final testing. Run the command
`npm run start` from the `bar` directory in one terminal. You should
see `express "bar" app listening on port 3000`. In another terminal
from the `foo` directory, run the command `npm run start`. You should
see `express "foo" app listening on port 3001`.

In the third terminal, run a few curl commands to confirm that
the services are successfully communicating with one another.

```bash
curl http://localhost:3000
# Hello from BAR
curl http://localhost:3001
# Hello from BAR from inside FOO
```

The *Foo* service on port 3001 is passing through the *Bar* service's
response, just as we asked it to. You can confirm this by inspecting
the console log of the *Foo* service. You should see:
`successfully received BAR response`.

## Conclusion

Let's wrap up this article for now. Next, we'll containerize these
two applications and test that their containers can
communicate with one another while running from docker. We'll use
docker-compose to start them together. In the third article, we'll
run the containers on kubernetes and have them communicate.
Hope to see you in the subsequent articles!
