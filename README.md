Socket.IO File Upload
=====================

This module provides functionality to upload files from a browser to a Node.JS server that runs Socket.IO.  Throughout the process, if their browser supports WebSockets, the user will not submit a single HTTP request.

The intended audience are single-page web apps, but other types of Node.JS projects may benefit from this library.

The module is released under the X11 open-source license.

## Quick Start

Navigate to your project directory and run:

    $ npm install socketio-file-upload

In your Express app, add the router like this (if you don't use Express, read the docs below):

```javascript
var siofu = require("socketio-file-upload");
var app = express()
    .use(siofu.router)
    .listen(8000);
```

On a server-side socket connection, do this:

```javascript
io.on("connection", function(socket){
    var uploader = new siofu();
    uploader.dir = "/path/to/save/uploads";
    uploader.listen(socket);
});
```

The client-side script is served at `/siofu/client.js`.  Include it like this:

```html
<script src="/siofu/client.js"></script>
```

The module also supports AMD; see the docs below for more information.

Then, in your client side app, with this HTML:

```html
<input type="file" id="siofu_input" />
```

Just do this in JavaScript:

```javascript
var socket = io.connect();
var uploader = new SocketIOFileUpload(socket);
uploader.listenOnInput(document.getElementById("siofu_input"));
```

That's all you need to get started.  For the detailed API, continue reading below.

## Table of Contents

- [Client-Side API](#client-side-api)
    - [instance.listenOnInput(input)](#instancelistenoninputinput)
    - [instance.listenOnDrop(element)](#instancelistenondropelement)
    - [instance.listenOnSubmit(submitButton, input)](#instancelistenonsubmitsubmitbutton-input)
    - [instance.listenOnArraySubmit(submitButton, input[])](#instancelistenonarraysubmitsubmitbutton-input)
    - [instance.prompt()](#instanceprompt)
    - [instance.submitFiles(files)](#instancesubmitfilesfiles)
    - [instance.destroy()](#instancedestroy)
    - [instance.useText = false](#instanceusetext--false)
    - [instance.serializeOctets = false](#instanceserializeoctets--false)
- [Client-Side Events](#events)
    - [choose](#choose)
    - [start](#start)
    - [load](#load)
    - [complete](#complete)
- [Server-Side API](#server-side-api)
    - [SocketIOFileUploadServer.listen(app)](#socketiofileuploadserverlistenapp)
    - [SocketIOFileUploadServer.router](#socketiofileuploadserverrouter)
    - [instance.listen(socket)](#instancelistensocket)
    - [instance.dir = "/path/to/upload/directory"](#instancedir--pathtouploaddirectory)
    - [instance.mode = "0666"](#instancemode--0666)
- [Server-Side Events](#events-1)
    - [start](#start-1)
    - [progress](#progress)
    - [complete](#complete-1)
    - [saved](#saved)
    - [error](#error)
- [Example](#example)

## Client-Side API

The client-side interface is inside the `SocketIOFileUpload` namespace.  Include it with:

```html
<script src="/siofu/client.js"></script>
```

If you're awesome and you use AMD/RequireJS, set up your paths config like this:

```javascript
requirejs.config({
    paths: {
        "SocketIOFileUpload": "/siofu/client",
        // ...
    }
});
```

and then include it in your app like this:

```javascript
define("app", ["SocketIOFileUpload"], function(SocketIOFileUpload){
    // ...
});
```

When instantiating an instance of the `SocketIOFileUpload`, pass a reference to your socket.

```javascript
var instance = new SocketIOFileUpload(socket);
```

### Public Properties and Methods

#### instance.listenOnInput(input)

When the user selects a file or files in the specified HTML Input Element, the library will begin to upload that file or those files.

JavaScript:

    instance.listenOnInput(document.getElementById("file_input"));

HTML:

    <label>Upload File: <input type="file" id="file_input" /></label>

All browsers tested support this method.

#### instance.listenOnDrop(element)

When the user drags and drops a file or files onto the specified HTML Element, the library will begin to upload that file or those files.

JavaScript:

    instance.listenOnDrop(document.getElementById("file_drop"));

HTML:

    <div id="file_drop">Drop Files Here</div>

In order to work, this method requires a browser that supports the HTML5 drag-and-drop interface.

#### instance.listenOnSubmit(submitButton, input)

Like `instance.listenOnInput(input)`, except instead of listening for the "change" event on the input element, listen for the "click" event of a button.

JavaScript:

    instance.listenOnSubmit(document.getElementById("my_button"), document.getElementById("file_input"));

HTML:

    <label>Upload File: <input type="file" id="file_input" /></label>
    <button id="my_button">Upload File</button>

#### instance.listenOnArraySubmit(submitButton, input[])

A shorthand for running `instance.listenOnSubmit(submitButton, input)` repeatedly over multiple file input elements.  Accepts an array of file input elements as the second argument.

#### instance.prompt()

When this method is called, the user will be prompted to choose a file to upload.

JavaScript:

    document.getElementById("file_button").addEventListener("click", instance.prompt, false);

HTML:

    <button id="file_button">Upload File</button>

Unfortunately, this method does not work in Firefox for security reasons.  Read the code comments for more information.

#### instance.submitFiles(files)

Call this method to manually submit a `FileList` object to be uploaded over the socket.  The argument is of type [FileList](https://developer.mozilla.org/en-US/docs/Web/API/FileList).

#### instance.destroy()

Unbinds all events and DOM elements created by this instance of SIOFU.

**Important Memory Note:** In order to remove the instance of SIOFU from memory, you need to do at least three things:

1. Remove all `siofu.prompt` event listeners *and then*
2. Call this function *and then*
3. Set this reference (and all references) to the instance to `null`

For example, if you created an instance like this:

    // ...
    var instance = new SocketIOFileUpload(socket);
    myBtn.addEventListener("click", instance.prompt, false);
    // ...

then you can remove it from memory like this:

    myBtn.removeEventListener("click", instance.prompt, false);
    instance.destroy();
    instance = null;

#### instance.useText = false

Defaults to `false`, which reads files as an octet array.  This is necessary for binary-type files, like images.

Set to `true` to read and transmit files as plain text instead.  This will save bandwidth if you expect to transmit only text files.  If you choose this option, it is recommended that you perform a filter by returning `false` to a `start` event if the file does not have a desired extension.

#### instance.useBuffer = false

Starting with Socket.IO 1.0, binary data may now be transmitted through the Web Socket.  You may tell SIOFU to transmit files as binary data by setting this option to `true`.  Defaults to `false`, which transmits files as base 64-encoded strings.

Advantages of enabling this option:

- Less overhead in the socket, since base 64 increases overhead by approximately 33%.
- No serialization and deserialization into and out of base 64 is required on the client and server side.

Disadvantages of enabling this option:

- Transmitting buffer types through a WebSocket is not supported in older browsers.
- This option is relatively new in both Socket.IO and Socket.IO File Upload and has not been rigorously tested.

As you use this option, [please leave feedback](https://github.com/vote539/socketio-file-upload/issues/16).  I'm hoping to enable this feature by default in a future version of Socket.IO File Upload.

#### instance.serializeOctets = false

*This method is experimental, and has been deprecated in Socket.IO File Upload as of version 0.3 in favor of instance.useBuffer.*

Defaults to `false`, which transmits binary files as Base 64 data (with a 33% overhead).

Set to `true` to instead transmit the data as a serialized octet array.  This will result in an overhead of over 1000% (not recommended for production applications).

*Note:* This option is not supported by Firefox.

### Events

Instances of the `SocketIOFileUpload` object implement the [W3C `EventTarget` interface](http://www.w3.org/wiki/DOM/domcore/EventTarget).  This means that you can do:

* `instance.addEventListener("type", callback)`
* `instance.removeEventListener("type", callback)`
* `instance.dispatchEvent(event)`

The events are documented below.

#### choose

The user has chosen files to upload, through any of the channels you have implemented.  If you want to cancel the upload, make your callback return `false`.

##### Event Properties

* `event.files` an instance of a W3C FileList object

#### start

This event is fired immediately following the `choose` event, but once per file.  If you want to cancel the upload for this individual file, make your callback return `false`.

##### Event Properties

* `event.file` an instance of a W3C File object

#### load

A file has been loaded into an instance of the HTML5 FileReader object and has been transmitted through Socket.IO.  We are awaiting a response from the server about whether the upload was successful; when we receive this response, a `complete` event will be dispatched.

##### Event Properties

* `event.file` an instance of a W3C File object
* `event.reader` an instance of a W3C FileReader object
* `event.name` the filename to which the server saved the file

#### complete

The server has received our file.

##### Event Properties

* `event.file` an instance of a W3C File object
* `event.success` true if the server-side implementation ran without error; false otherwise
* `event.detail` The value of `file.clientDetail` on the server side.  Properties may be added to this object literal during any event on the server side.

## Server-Side API

The server-side interface is contained within an NPM module.  Require it with:

    var SocketIOFileUploadServer = require("socketio-file-upload");

### Static Properties and Methods

#### SocketIOFileUploadServer.listen(app)

If you are using an HTTP server in Node, pass it into this method in order for the client-side JavaScript file to be served.

    var app = http.createServer( /* your configurations here */ ).listen(80);
    SocketIOFileUploadServer.listen(app);

#### SocketIOFileUploadServer.router

If you are using Connect-based middleware like Express, pass this value into the middleware.

    var app = express()
                .use(SocketIOFileUploadServer.router)
                .use( /* your other middleware here */ )
                .listen(80);

### Public Properties and Methods

#### instance.listen(socket)

Listen for uploads occuring on this Socket.IO socket.

    io.sockets.on("connection", function(socket){
        var uploader = new SocketIOFileUploadServer();
        uploader.listen(socket);
    });

#### instance.dir = "/path/to/upload/directory"

If specified, the module will attempt to save uploaded files in this directory.  The module will inteligently suffix numbers to the uploaded filenames until name conflicts are resolved.  It will also sanitize the filename to help prevent attacks.

The last-modified time of the file might be retained from the upload.  If this is of high importance to you, I recommend performing some tests, and if it does not meet your needs, submit an issue or a pull request.

#### instance.mode = "0666"

Use these UNIX permissions when saving the uploaded file.  Defaults to `0666`.

### Events

Instances of `SocketIOFileUploadServer` implement [Node's `EventEmitter` interface](http://nodejs.org/api/events.html#events_class_events_eventemitter).  This means that you can do:

* `instance.on("type", callback)`
* `instance.removeListener("type", callback)`
* `instance.emit("type", event)`
* et cetera.

The events are documented below.

#### start

The client has started the upload process.

##### Event Properties

* `event.file` An object containing the file's `name`, `mtime`, `encoding`, `meta`, and `id`.
    *Note:* `encoding` is either "text" if the file is being transmitted as plain text or "octet" if it is being transmitted using an ArrayBuffer.  *Note:* In the "progress", "complete", "saved", and "error" events, if you are letting the module save the file for you, the file object will contain two additional properties: `base`, the new base name given to the file, and `pathName`, the full path at which the uploaded file was saved.

#### progress

Data has been received from the client.

##### Event Properties

* `event.file` The same file object that would have been passed during the `start` event earlier.
* `event.buffer` A buffer containing the data received from the client

#### complete

The transmission of a file is complete.

##### Event Properties

* `event.file` The same file object that would have been passed during the `start` event earlier.
* `event.interrupt` true if the client said that the data was interrupted (not completely sent); false otherwise

#### saved

A file has been successfully saved.

##### Event Properties

* `event.file` The same file object that would have been passed during the `start` event earlier.

#### error

An error was encountered in the saving of the file.

##### Event Properties

* `event.file` The same file object that would have been passed during the `start` event earlier.
* `event.error` The I/O error that was encountered.

## Adding Meta Data

It is sometimes useful to add metadata to a file prior to uploading the file.  You may add metadata to a file on the client side by setting the `file.meta` property on the File object during the "choose" or "start" events.  You may also add metadata to a file on the server side by setting the `file.clientDetail` property on the fileInfo object during any of the server-side events.

### Client to Server Meta Data

To add meta data to an individual file, you can listen on the "start" event as shown below.

```javascript
// client side
siofu.addEventListener("start", function(event){
    event.file.meta.hello = "world";
});
```

The data is then available on the server side as follows.

```javascript
// server side
uploader.on("saved", function(event){
    console.log(event.file.meta.hello);
});
```

You can also refer back to your meta data at any time on the client side by referencing the same `event.file.meta` object literal.

### Server to Client Meta Data

You can add meta data on the server.  The meta data will be available to the client on the "complete" event on the client as shown below.

```javascript
// server side
siofuServer.on("saved", function(event){
    event.file.clientDetail.hello = "world";
});
```

The information saved in `event.file.clientDetail` will be available in `event.detail` on the client side.

```javascript
// client side
siofu.addEventListener("complete", function(event){
    console.log(event.detail.hello);
});
```

## Example

This example assumes that you are running your application via the Connect middleware, including Express.  If you are using a middleware that is not Connect-based or Node-HTTP-based, download the `client.js` file from the project repository and serve it on the path `/siofu/client.js`.  Alternatively, you may contribute an adapter for your middleware to this project and submit a pull request.

### Server Code: app.js

    // Require the libraries:
    var SocketIOFileUploadServer = require('socketio-file-upload'),
        socketio = require('socket.io'),
        express = require('express');

    // Make your Express server:
    var app = express()
        .use(SocketIOFileUploadServer.router)
        .use(express.static(__dirname + "/public"))
        .listen(80);

    // Start up Socket.IO:
    var io = socketio.listen(app);
    io.sockets.on("connection", function(socket){

        // Make an instance of SocketIOFileUploadServer and listen on this socket:
        var uploader = new SocketIOFileUploadServer();
        uploader.dir = "/srv/uploads";
        uploader.listen(socket);

        // Do something when a file is saved:
        uploader.on("saved", function(event){
            console.log(event.file);
        });

        // Error handler:
        uploader.on("error", function(event){
            console.log("Error from uploader", event);
        });
    });

### Client Code: public/index.html

    <!DOCTYPE html>
    <html>
    <head>
    <title>Upload Files</title>
    <script src="/siofu/client.js"></script>
    <script src="/socket.io/socket.io.js"></script>

    <script type="text/javascript">
    document.addEventListener("DOMContentLoaded", function(){

        // Initialize instances:
        var socket = io.connect();
        var siofu = new SocketIOFileUpload(socket);

        // Configure the three ways that SocketIOFileUpload can read files:
        document.getElementById("upload_btn").addEventListener("click", siofu.prompt, false);
        siofu.listenOnInput(document.getElementById("upload_input"));
        siofu.listenOnDrop(document.getElementById("file_drop"));

        // Do something when a file is uploaded:
        siofu.addEventListener("complete", function(event){
            console.log(event.success);
            console.log(event.file);
        });

    }, false);
    </script>

    </head>
    <body>

    <p><button id="upload_btn">Prompt for File</button></p>
    <p><label>Choose File: <input type="file" id="upload_input"/></label></p>
    <div id="file_drop" dropzone="copy" title="drop files for upload">Drop File</div>

    </body>
    </html>

## Future Inovations

I hope to one day see this project implement the following features.

* Upload Progress.  There are examples of this on the net, so it should be feasible to implement.
* Allow input of a file URL rather than uploading a file from your computer or mobile device.
