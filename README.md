Description
===========

promise-ftp is an FTP client module for [node.js](http://nodejs.org/) that provides an asynchronous interface for
communicating with an FTP server.

This module is a wrapper around the [node-ftp](https://github.com/mscdex/node-ftp) module, and provides some additional
features as well as a convenient promise-based API.

This library is written primarily in CoffeeScript, but may be used just as easily in a Node app using Javascript or
CoffeeScript.  Promises in this module are provided by [Bluebird](https://github.com/petkaantonov/bluebird).


Change Log
============

* Version 1.3.0 changes:
  * addition of the **rawClient** property which exposes the underlying ftp client.

* Version 1.2.0 changes:
  * the `FtpConnectionError` and `FtpReconectError` classes have been moved to
  [their own module](https://github.com/realtymaps/promise-ftp-errors), in anticipation of creating a
  semi-interchangeable API for [promise-sftp](https://github.com/realtymaps/promise-sftp).  They are still exposed
  on the main module, but for npm 3+ the `promise-ftp-errors` module must now be listed as an explicit dependency.

* Version 1.1.0 adds:
  * the `autoReconnect` and `preserveCwd` options
  * the `FtpReconectError` error class
  * the `reconnect()` and `getConnectionStatus()` methods
  * the `STATUSES` and `ERROR_CODES` maps
  * completed API documentation

* Version 1.0.0 provides a promise-based API that mirrors the
[node-ftp API](https://github.com/mscdex/node-ftp/blob/master/README.md#api) almost identically, except with a promise
interface, plus:
  * the `FtpConnectionError` error class


Requirements
============

* [node.js](http://nodejs.org/) -- v0.8.0 or newer


Install
=======

    npm install promise-ftp
    npm install promise-ftp-common   # only necessary on npm 3+


Examples
========

Get a directory listing of the current (remote) working directory:

```javascript
  var PromiseFtp = require('promise-ftp');
  
  var ftp = new PromiseFtp();
  ftp.connect({host: host, user: user, password: password})
  .then(function (serverMessage) {
    console.log('Server message: '+serverMessage);
    return ftp.list('/');
  }).then(function (list) {
    console.log('Directory listing:');
    console.dir(list);
    return ftp.end();
  });
```

Download remote file 'foo.txt' and save it to the local file system:

```javascript
  var PromiseFtp = require('promise-ftp');
  var fs = require('fs');
  
  var ftp = new PromiseFtp();
  ftp.connect({host: host, user: user, password: password})
  .then(function (serverMessage) {
    return ftp.get('foo.txt');
  }).then(function (stream) {
    return new Promise(function (resolve, reject) {
      stream.once('close', resolve);
      stream.once('error', reject);
      stream.pipe(fs.createWriteStream('foo.local-copy.txt'));
    });
  }).then(function () {
    return ftp.end();
  });
```

Upload local file 'foo.txt' to the server:

```javascript
  var PromiseFtp = require('promise-ftp');
  var fs = require('fs');
  
  var ftp = new PromiseFtp();
  ftp.connect({host: host, user: user, password: password})
  .then(function (serverMessage) {
    return ftp.put('foo.txt', 'foo.remote-copy.txt');
  }).then(function () {
    return ftp.end();
  });
```


API
===

For the most part, this module's API mirrors [node-ftp's API](https://github.com/mscdex/node-ftp#api), except that it
returns promises which resolve or reject, rather than emitting events or calling callbacks.  However, there are some
minor differences and some additional features. If you need access to the underlying events or callback-based methods,
you can access the raw node-ftp client via **rawClient** property.

Errors
------

Errors generated by this module will be instances of one of the following:

* **FtpConnectionError**: Indicates the connection status was not appropriate for the method called; e.g. **connect**()
or **reconnect()** was called when the connection was not in a disconnected state, **end()** was called when the
connection has already closed, etc.

* **FtpReconnectError**: Only possible when the `autoReconnect` option is set true, this indicates a reconnect was
attempted and failed.  It includes information about both the original disconnect error (if any) as well as the error
while attempting to reconnect.

In addition, errors from the underlying node-ftp library may be rejected unchanged through method calls.  In the case
of protocol-level errors, the rejected error will contain a `code` property that references the related 3-digit FTP
response code.  This code may be translated into a (generic) human-readable text explanation by referencing the map
`PromiseFtp.ERROR_CODES`.

Methods
-------

* **\[constructor\]**(): Creates and returns a new PromiseFtp client instance.

* **connect**(config <_object_>): Connects to an FTP server; returned promise resolves to the server's greeting
message. Valid config properties:

    * host <_string_>: The hostname or IP address of the FTP server. **Default:** 'localhost'

    * port <_integer_>: The port of the FTP server. **Default:** 21

    * secure <_mixed_>: Set to true for both control and data connection encryption, 'control' for control connection
    encryption only, or 'implicit' for implicitly encrypted control connection (this mode is deprecated in modern times,
    but usually uses port 990) **Default:** false

    * secureOptions <_object_>: Additional options to be passed to `tls.connect()`. **Default:** (none)

    * user <_string_>: Username for authentication. **Default:** 'anonymous'

    * password <_string_>: Password for authentication. **Default:** 'anonymous@'

    * connTimeout <_integer_>: How long (in milliseconds) to wait for the control connection to be established.
    **Default:** 10000

    * pasvTimeout <_integer_>: How long (in milliseconds) to wait for a PASV data connection to be established.
    **Default:** 10000

    * keepalive <_integer_>: How often (in milliseconds) to send a 'dummy' (NOOP) command to keep the connection alive.
    **Default:** 10000
    
    * autoReconnect <_boolean_>: Whether to attempt to automatically reconnect using the existing config if the
    connection is unexpectedly closed.  Auto-reconnection is lazy, and so will wait until a command needs to be issued
    before attempting to reconnect.  **Default:** false
    
    * preserveCwd <_boolean_>: Whether to attempt to return to the prior current working directory after a successful
    automatic reconnection.  Only used if `autoReconnect` is true.  **Default:** false
    
* **reconnect**(): Connects to an FTP server using the config from the most recent call to **connect()**.  Returned
promise resolves to the server's greeting message.

* **end**(): Closes the connection to the server after any/all enqueued commands have been executed; returned promise
resolves to any error associated with closing the connection, or _true_ if there was an error but it wasn't captured.

* **destroy**(): Closes the connection to the server immediately.  Returns a boolean indicating whether the connection
was connected prior to the call to **destroy()**.

* **getConnectionStatus**(): Returns a string describing the current connection state.  Possible strings are
enumerated in `PromiseFtp.`STATUSES, as well as below:

    * not yet connected
    
    * connecting
    
    * connected
    
    * logging out
    
    * disconnecting
    
    * disconnected
    
    * reconnecting

### Required "standard" commands (RFC 959)

* **list**(\[path <_string_>\]\[, useCompression <_boolean_>\]): Retrieves the directory listing of `path`. `path`
defaults to the current working directory. `useCompression` defaults to false.  Returned promise resolves to an array
of objects with these properties:

      * type <_string_>: A single character denoting the entry type: 'd' for directory, '-' for file (or 'l' for
      symlink on **\*NIX only**).

      * name <_string_>: The name of the entry.

      * size <_string_>: The size of the entry in bytes.

      * date <_Date_>: The last modified date of the entry.

      * rights <_object_>: The various permissions for this entry **(*NIX only)**.

          * user <_string_>: An empty string or any combination of 'r', 'w', 'x'.

          * group <_string_>: An empty string or any combination of 'r', 'w', 'x'.

          * other <_string_>: An empty string or any combination of 'r', 'w', 'x'.
     
      * owner <_string_>: The user name or ID that this entry belongs to **(*NIX only)**.

      * group <_string_>: The group name or ID that this entry belongs to **(*NIX only)**.

      * target <_string_>: For symlink entries, this is the symlink's target **(*NIX only)**.

      * sticky <_boolean_>: True if the sticky bit is set for this entry **(*NIX only)**.

* **get**(path <_string_>\[, useCompression <_boolean_>\]): Retrieves a file at `path` from the server.
`useCompression` defaults to false. Returned promise resolves to a `ReadableStream`.

* **put**(input <_mixed_>, destPath <_string_>\[, useCompression <_boolean_>\]): Sends data to the server to be stored
as `destPath`. `input` can be a ReadableStream, a Buffer, or a path to a local file. `useCompression` defaults to
false. Returned promise resolves to _undefined_.

* **append**(input <_mixed_>, destPath <_string_>\[, useCompression <_boolean_>\]): Same as **put()**, except if
`destPath` already exists, it will be appended to instead of overwritten.

* **rename**(oldPath <_string_>, newPath <_string_>): Renames/moves `oldPath` to `newPath` on the server. Returned
promise resolves to _undefined_.

* **logout**(): Logs the user out from the server. Returned promise resolves to _undefined_.

* **delete**(path <_string_>): Deletes the file at `path`. Returned promise resolves to _undefined_.

* **cwd**(path <_string_>): Changes the current working directory to `path`. Returned promise resolves to the new
current directory, if the server replies with it in the response text; otherwise resolves to _undefined_.

* **abort**(): Aborts the current data transfer (e.g. from **get()**, **put()**, or **list()**). Returned promise
resolves to _undefined_.

* **site**(command <_string_>): Sends `command` (e.g. 'CHMOD 755 foo', 'QUOTA') using SITE; returned promise resolves
to an object with the following attributes:

  * text <_string_>: responseText
  
  * code <_integer_>: responseCode.

* **status**(): Retrieves human-readable information about the server's status. Returned promise resolves to the status
string sent by the server.

* **ascii**(): Sets the transfer data type to ASCII. Returned promise resolves to _undefined_.

* **binary**(): Sets the transfer data type to binary (default at time of connection). Returned promise resolves to
_undefined_.

### Optional "standard" commands (RFC 959)

* **mkdir**(path <_string_>\[, recursive <_boolean_>\]): Creates a new directory, `path`, on the server. `recursive` is
for enabling a 'mkdir -p' algorithm and defaults to false. Returned promise resolves to _undefined_.

* **rmdir**(path <_string_>\[, includeContents <_boolean_>\]): Removes a directory, `path`, on the server. If
`includeContents` is true, this call will delete the contents of the directory if it is not empty; note that this
currently only deletes files within the directory, not subdirectories (the command will fail if there are subdirectories
present). Returned promise resolves to _undefined_.

* **cdup**(): Changes the working directory to the parent of the current directory. Returned promise resolves to
_undefined_.

* **pwd**(): Retrieves the current working directory. Returned promise resolves to the current working directory.

* **system**(): Retrieves the server's operating system. Returned promise resolves to the OS string sent by the server.

* **listSafe**(\[path <_string_>\]\[, useCompression <_boolean_>\]): Similar to **list()**, except the directory is
temporarily changed to `path` to retrieve the directory listing. This is useful for servers that do not handle
characters like spaces and quotes in directory names well for the LIST command. This function is "optional" because it
relies on **pwd()** being available.

### Extended commands (RFC 3659)

* **size**(path <_string_>): Retrieves the size of `path`. Returned promise resolves to number of bytes.

* **lastMod**(path <_string_>): Retrieves the last modified date and time for `path`. Returned promise resolves an
instance of `Date`.

* **restart**(byteOffset <_integer_>): Sets the file byte offset for the next file transfer action initiated via
**get()** or **put()** to `byteOffset`. Returned promise resolves to _undefined_.
