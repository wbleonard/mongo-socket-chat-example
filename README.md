# Integrating MongoDB Change Streams with Socket.IO

# Introduction

The [Socket.IO Getting Started guide](https://socket.io/get-started/chat) provides a nice introduction to [Socket.io](https://socket.io/). The guide bundles the server and client into a single application where messages submitted via an HTML input form are received and displayed on the page. 

Since MongoDB supports an exceptional [eventing framwork](https://www.mongodb.com/docs/manual/changeStreams/) of its own, this tutorial will demonstrate how to propogate events emmitted from MongoDB through to Socket.IO. To keep things consistent, I will try to mirror the Socket.IO Getting Started guide as much as possible. Let's get started...

# The web framework

As with the Socket.IO getting started guide, we're going to set up a simple HTML webpage, however, in our example, it's only going to display a list of messages -- there will be no input form.

First let's create a `package.json` manifest file that describes our project. I recommend you place it in a dedicated empty directory (I'll call mine `mongo-socket-chat-example`).


```JSON
{
 "name": "monngo-socket-chat-example",
 "version": "0.0.1",
 "description": "my first mongo socket.io app",
 "dependencies": {}
}
```

Then use `npm` to install `express`:
```zsh
npm install express@4
```

One express is installed, we can set up an `index.js` file that will set up our application.


```JavaScript
const express = require('express');
const app = express();
const http = require('http');
const server = http.createServer(app);

app.get('/', (req, res) => {
 res.send('<h1>Hello world</h1>');
});

server.listen(3000, () => {
 console.log('listening on *:3000');
});
```


This means that:



* Express initializes `app` to be a function handler that you can supply to an HTTP server (as seen in line 4).
* We define a route handler / that gets called when we hit our website home.
* We make the http server listen on port 3000.

If you run `node index.js` you should see the following:

![node index.js](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/run1.png "node index.js")

And if you point your browser to [`http://localhost:3000`](http://localhost:3000):

![http://localhost:3000](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/result1.png "http://localhost:3000")

# Serving HTML

So far in `index.js` we are calling `res.send` and passing it a string of HTML. Our code would look very confusing if we just placed our entire application’s HTML there, so instead we're going to create an `index.html` file and serve that instead.

Let’s refactor our route handler to use `sendFile` instead.

```JavaScript
app.get('/', (req, res) => {

 res.sendFile(__dirname + '/index.html');

});
```

Here's a simple `index.html` file to display a list of messages with some styling included:

```HTML
<!DOCTYPE html>
<html>
  <head>
    <title>Socket.IO chat</title>
    <style>
      body { margin: 0; padding-bottom: 3rem; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }

      #messages { list-style-type: none; margin: 0; padding: 0; }
      #messages > li { padding: 0.5rem 1rem; }
      #messages > li:nth-child(odd) { background: #efefef; }
    </style>
  </head>
  <body>
    <h1>Hello world from file</h1>
    <ul id="messages"></ul>
</body>
</html>

```

If you restart the process (by hitting Control+C and running `node index.js` again) and refresh the page it should look like this:

![node index.js](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/result2.png "node index.js")

# Integrating MongoDB

Since you're here, I'm going to assume you already have access to a MongoDB cluster. If you do not, just follow these [Create a MongoDB Cluster](https://www.mongodb.com/docs/drivers/node/current/quick-start/#create-a-mongodb-cluster) instructions.

   > Note, a cluster (replica set) is required because we'll be using [change streams](https://www.mongodb.com/docs/manual/changeStreams/), which require an [oplog](https://www.mongodb.com/docs/manual/core/replica-set-oplog/). There's no need to worry, however, as it is easy enough to [configure a single node replica set](https://www.mongodb.com/docs/manual/tutorial/convert-standalone-to-replica-set/). 


## Create the Database and Collection

For this example I'm going to create a `chat` database with a `messages` collection along with an initial record that I will later use to validate connectivity to MongoDB from my client application: 

![message record](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/message.png "message record")

## Install dotenv

To avoid storing MongoDB credentials in our application code, we'll use [dotenv](https://www.npmjs.com/package/dotenv) to read the MongoDB connection string from our environment. As with the `express` framework, use npm to install `dotenv`:

```zsh
npm install dotenv --save
```

Create a `.env` file with the following `MONGODB_CONNECTION_STRING` variable:

```JavaScript
MONGODB_CONNECTION_STRING='<Your MongoDB Connection String>' 
```

Then add the following to your `index.js`:

```JavaScript
require('dotenv').config()
console.log(process.env.MONGODB_CONNECTION_STRING) // remove this after you've confirmed it working
```

If you restart the process (by hitting Control+C and running `node index.js` again) you can verify that your environment is working properly:

![run2](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/run3.png "run3")



## Install the Node.js driver:

Use npm once again to install the Node.js driver: 

```zsh
npm install mongodb@4.5
```
## Connect to MongoDB

Add the following code to your `index.js`:

```JavaScript
const { MongoClient } = require("mongodb");

const client = new MongoClient(process.env.MONGODB_CONNECTION_STRING);  // remove this after you've confirmed it working

async function run() {

  try {

    await client.connect();
    const database = client.db('chat');
    const messages = database.collection('messages');

    // Query for our test message:
    const query = { message: 'Hello from MongoDB' };
    const message = await messages.findOne(query);
    console.log(message);

  } catch {

    // Ensures that the client will close when you error
    await client.close();
  }
}

run().catch(console.dir);
```
Restart your application and you should see the following

![run4](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/run4.png "run4")

For further information, this [MongoDB Node.js Quick Start](https://www.mongodb.com/docs/drivers/node/current/quick-start/) provides an excellent introduction to incorporating MongoDB into your Node.js applications.  

## Watching for Changes
We want to be alerted any time a new message is inserted into the database. For the purpose of this tutorial we'll also [watch](https://www.mongodb.com/docs/drivers/node/current/usage-examples/changeStream/) for message updates. Replace the three lines of query test code in `index.js` with the following:

```JavaScript
       // open a Change Stream on the "messages" collection
       changeStream = messages.watch();

       // set up a listener when change events are emitted
       changeStream.on("change", next => {
           // process any change event
           switch (next.operationType) {
               case 'insert':
                   console.log(next.fullDocument.message);
                   break;
               case 'update':
                   console.log(next.updateDescription.updatedFields.message);
           }
       });
```

Then edit and/or insert some messages: 

![run5](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/run5.png "run5")

# Integrating Socket.IO
Socket.IO is composed of two parts:



* A server that integrates with (or mounts on) the Node.JS HTTP Server [socket.io](https://github.com/socketio/socket.io)
* A client library that loads on the browser side [socket.io-client](https://github.com/socketio/socket.io-client)

During development, `socket.io` serves the client automatically for us, as we’ll see, so for now we only have to install one module:

```zsh
npm install socket.io
```

That will install the module and add the dependency to `package.json`. Now let’s edit `index.js` to add it:

```JavaScript
const { Server } = require("socket.io");
const io = new Server(server);
```

Now in `index.html` add the following snippet before the `</body>` (end body tag):

```HTML
<script src="/socket.io/socket.io.js">&lt;/script>
<script>
   var socket = io();
</script>
```

## Broadcasting

The next goal is for us to emit the event from the server to the rest of the users.

In order to send an event to everyone, Socket.IO gives us the `io.emit()` method, which looks as follows:

```JavaScript
io.emit('<event name>', '<event data>')
```

So, augment our change stream code with the following: 

```JavaScript
      switch (next.operationType) {
                case 'insert':
                    io.emit('chat message', next.fullDocument.message);
                    console.log(next.fullDocument.message);
                    break;

                case 'update':
                    io.emit('chat message', next.updateDescription.updatedFields.message);
                    console.log(next.updateDescription.updatedFields.message);
            }
```

And on the client side when we capture a 'chat message' event we’ll include it in the page. The _total_ client-side JavaScript code now amounts to:

```HTML
<!DOCTYPE html>
<html>
 <head>
   <title>Socket.IO chat</title>
   <style>
     body { margin: 0; padding-bottom: 3rem; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }

     #messages { list-style-type: none; margin: 0; padding: 0; }
     #messages > li { padding: 0.5rem 1rem; }
     #messages > li:nth-child(odd) { background: #efefef; }
   </style>
 </head>
 <body>
   <ul id="messages"></ul>

   <script src="/socket.io/socket.io.js"></script>
   <script>
       var socket = io();
       var messages = document.getElementById('messages');

       socket.on('chat message', function(msg) {
           var item = document.createElement('li');
           item.textContent = msg;
           messages.appendChild(item);
           window.scrollTo(0, document.body.scrollHeight);
       });
   </script>
</body>
</html>
```


And that completes our chat application, in about 80 lines of code! This is what it looks like on the web client when messages are inserted or updated in our `chat.messages` collection in MongoDB:

![result-final](https://github.com/wbleonard/monngo-socket-chat-example/blob/main/images/result-final.png "result-final")

