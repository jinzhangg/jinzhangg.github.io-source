SockJS, Redis, Node.js Tutorial
###################################

:date: 2013-04-18
:author: Jin Zhang

I became interested in real-time web applications when I saw an excellent `PyCon video <http://pyvideo.org/video/1798/make-more-responsive-web-applications-with-socket>`_ on Socket.io by Luke Sneeringer. I decided to explore the subject and was disappointed to find little to none easy to follow tutorials available on the subject. This is not surprising given that WebSockets is still in its early stages. There are still issues dealing with proper browser support for the WebSockets standard. Libraries such as socket.io and SockJS aim to fix this problem by falling back on less optimal but more widely support methods to mimic two way communication for example by using Flash or long polling and others.

In this tutorial, I will describe how to architect a real-time chat web application using SockJS, Redis, and node.js.

Why use SockJS over socket.io?
Socket.io is certainly the more popular choice with 8k stars on Github compared to 1.2k for SockJS. The reader can investigate both frameworks and decide for himself which to use but during my research here are the main reasons I chose SockJS over socket.io:

    1. `Discussion <https://groups.google.com/forum/#!msg/sockjs/lgzxVnlth54/NbQKNEAzB5cJ>`_ from one of the main SockJS contributors comparing the two frameworks.
    2. `PythonAnywhere <http://blog.pythonanywhere.com/27/>`_ blog post about benchmarking between the two libraries.
    3. This is related to the Python gevent-socketio library. Since I'm a Python user, this was the implementation I would use if I selected socketio. `Mailing list <https://groups.google.com/d/msg/gevent-socketio/lHhaUZu7tTo/ZRBcohD9A7gJ>`_ on April 15, 2013 discussing a possible memory leak which author acknowledges.

We will be using the `Express <http://expressjs.com/>`_ microframework as a layer over Node. Express provides some background lifting which will help us write a typical web application easier. I am only using Express in order to keep this tutorial as simple as possible. In a production setting, it is a common idea to use the web framework of your choice (Django, Flask, Rails, etc.) to handle the non-websockets section of your website and only use node.js with SockJS as necessary to provide real time functionality.

We will be extending one of the official SockJS chat server examples to use the Redis pub/sub system which will be closer to a real deployment situation. The pub/sub system is used to relay messages between the server and clients.

I will be using a Ubuntu 12.04 server from `DigitalOcean <https://www.digitalocean.com/?refcode=34ed21971862>`_. On a fresh machine do the following:

.. code-block:: javascript

    $ sudo apt-get update
    $ sudo apt-get install build-essential

Install packages we will be using:

.. code-block:: javascript

    # Install package required to use add-apt-repository
    $ sudo apt-get install python-software-properties
    # Node.js 0.10.4
    $ sudo add-apt-repository ppa:chris-lea/node.js
    # Redis-server 2.6.12
    $ sudo add-apt-repository ppa:chris-lea/redis-server
    # Update apt-get
    $ sudo apt-get update
    # Now install our packages
    $ sudo apt-get install nodejs redis-server

Create a directory called :code:`sockjs-tutorial` to hold our project. Inside the directory create a file called :code:`package.json`. This is a node.js specific file containing details about which package we will be using.

.. code-block:: javascript

    {
        "name": "sockjs-express",
        "version": "0.0.0-unreleasable",
        "dependencies": {
            "express": "~3*",
            "sockjs": "*",
            "hiredis": "*",
            "redis": "*"
        }
    }

Now we can install them using this command: :code:`$ npm install`

Next create a file called server.js which will contain our node.js server code.
Inside the file, declare which packages we will be using:

.. code-block:: javascript

    var express = require('express');
    var sockjs  = require('sockjs');
    var http    = require('http');
    var redis   = require('redis');

Create one publisher Redis client where we will push all user messages to.

.. code-block:: javascript

    var publisher = redis.createClient();

Create the Sockjs server

.. code-block:: javascript

    var sockjs_opts = {sockjs_url: "http://cdn.sockjs.org/sockjs-0.3.min.js"};
    var sockjs_chat = sockjs.createServer(sockjs_opts);

Next is where the real work happens.

.. code-block:: javascript

    sockjs_chat.on('connection', function(conn) {
        var browser = redis.createClient();
        browser.subscribe('chat_channel');

        // When we see a message on chat_channel, send it to the browser
        browser.on("message", function(channel, message){
            conn.write(message);
        });

        // When we receive a message from browser, send it to be published
        conn.on('data', function(message) {
            publisher.publish('chat_channel', message);
        });
    });

Node.js is based on event programming meaning certain actions will emit an event when they trigger. One of the events is called :code:`connection`. This will trigger when a browser client makes the initial connection to our node.js server. When this event triggers, we want to execute the anonymous function inside giving us a handle to the connection object called :code:`conn`.

When the browser client connects, we want to create a new Redis client which will subscribe to our publisher we created earlier using the channel :code:`chat_channel`.

.. code-block:: javascript

    browser.on("message", function(channel, message){
        conn.write(message);
    });

This part of the code is the Redis subscriber client. When the channel we're subscribed to has a new message event, we want to grab that message and send it back to the browser using the SockJS :code:`conn` object.

.. code-block:: javascript

    conn.on('data', function(message) {
            publisher.publish('chat_channel', message);
        });

Now we're coding on the connection object and when the connection receives data from the browser client, we're going to publish it to our Redis publisher on the channel :code:`chat_channel`. When we publish it to the channel, all current subscribers to that channel will trigger their :code:`on("message")` event. In our code, we want that event to also send a message to browser containing the message.

The last part of our code simply contains the Express and node.js framework boilerplate code which we need to serve a HTML file.

.. code-block:: javascript

    // Express server
    var app = express();
    var server = http.createServer(app);

    sockjs_chat.installHandlers(server, {prefix:'/chat'});

    console.log(' [*] Listening on 0.0.0.0:9001' );
    server.listen(9001, '0.0.0.0');

    app.get('/', function (req, res) {
        res.sendfile(__dirname + '/index.html');
    });

The one SockJS part in there is the :code:`sockjs_chat.installHandlers` code. We are simply binding our SockJS code to the server using the prefix :code:`/chat`. We will need to use this prefix in our browser client javascript code which we will write next. The final server.js file should look like below:

.. code-block:: javascript

    var express = require('express');
    var sockjs  = require('sockjs');
    var http    = require('http');
    var redis   = require('redis');


    // Redis publisher
    var publisher = redis.createClient();

    // Sockjs server
    var sockjs_opts = {sockjs_url: "http://cdn.sockjs.org/sockjs-0.3.min.js"};
    var sockjs_chat = sockjs.createServer(sockjs_opts);
    sockjs_chat.on('connection', function(conn) {
        var browser = redis.createClient();
        browser.subscribe('chat_channel');

        // When we see a message on chat_channel, send it to the browser
        browser.on("message", function(channel, message){
            conn.write(message);
        });

        // When we receive a message from browser, send it to be published
        conn.on('data', function(message) {
            publisher.publish('chat_channel', message);
        });
    });

    // Express server
    var app = express();
    var server = http.createServer(app);

    sockjs_chat.installHandlers(server, {prefix:'/chat'});

    console.log(' [*] Listening on 0.0.0.0:9001' );
    server.listen(9001, '0.0.0.0');

    app.get('/', function (req, res) {
        res.sendfile(__dirname + '/index.html');
    });

Now create a file called index.html and copy-paste the following code:

.. code-block:: html

    <!doctype html>
    <html><head>
        <script src="http://ajax.googleapis.com/ajax/libs/jquery/1.7.1/jquery.min.js"></script>
        <script src="http://cdn.sockjs.org/sockjs-0.3.min.js"></script>
        <style>
            .box {
                width: 300px;
                float: left;
                margin: 0 20px 0 20px;
            }
            .box div, .box input {
                border: 1px solid;
                -moz-border-radius: 4px;
                border-radius: 4px;
                width: 100%;
                padding: 0px;
                margin: 5px;
            }
            .box div {
                border-color: grey;
                height: 300px;
                overflow: auto;
            }
            .box input {
                height: 30px;
            }
            h1 {
                margin-left: 30px;
            }
            body {
                background-color: #F0F0F0;
                font-family: "Arial";
            }
        </style>
    </head><body lang="en">
    <h1>SockJS, Redis, Node.js Tutorial</h1>

    <div id="first" class="box">
        <div></div>
        <form><input autocomplete="off"></input></form>
    </div>

    <script>
        var sockjs_url = '/chat';
        var sockjs = new SockJS(sockjs_url);

        var userid = 'guest' + new Date().getSeconds();
        var div  = $('#first div');
        var inp  = $('#first input');
        var form = $('#first form');

        var print = function(message){
            div.append($("<code>").text(message));
            div.append($("<br>"));
            div.scrollTop(div.scrollTop()+10000);
        }

        sockjs.onopen    = function()  {print('Connected.');};
        sockjs.onmessage = function(e) {print(e.data);};
        sockjs.onclose   = function()  {print('Closing Connection.');};

        form.submit(function() {
            print('Sending to server...');
            sockjs.send(userid + ': ' + inp.val());
            inp.val('');
            return false;
        });

    </script>
    </body></html>

The SockJS part that we are interested in is the three lines below:

.. code-block:: javascript

    sockjs.onopen    = function()  {print('Connected.');};
    sockjs.onmessage = function(e) {print(e.data);};
    sockjs.onclose   = function()  {print('Closing Connection.');};

These are three events that SockJS will emit during the lifetime of the connection. When the connection is first established between the SockJS browser client and the Node.js server, it will trigger the :code:`sockjs.onopen` function. Any messages sent from the server to the browser during this connection will trigger the :code:`sockjs.onmessage` function, which we simply ask Javascript to print to our browser. Finally, when the connection is closed, we will trigger the :code:`sockjs.onclose` function.

The last piece to discuss is sending data from the browser back to the server. This work is done in this function:

.. code-block:: javascript

    form.submit(function() {
        print('Sending to server...');
        sockjs.send(userid + ': ' + inp.val());
        inp.val('');
        return false;
    });

We simply call :code:`sockjs.send("some message here")` in order to send data back to Node.js. In our case, we send a variable called userid which we simply define as a "guest" string plus a random number using the current time in seconds. We add this to the form's input value to complete the message.

Now for the fun part. Start our node.js server by entering the command into your terminal:

.. code-block:: javascript

    $ node server.js

Open up two browsers and navigate to http://localhost:9001. When you type in a message from one browser, the second browser will receive the message almost instantaneously!

That's it. We have just made a real-time web application that can utilize a two-way communication link between the server and browser. I think that's pretty cool. This is the future of the web and just as we moved from static HTML pages to dynamic database driven websites, we will move towards real-time web applications.

In the future, I will make another post describing how to put everything together and deploy using Nginx and either Flask or Django as the main web framework with node.js only handling the WebSockets section of our website.

You can view the source code on `GitHub <https://github.com/jinzhangg/sockjs-tutorial>`_.