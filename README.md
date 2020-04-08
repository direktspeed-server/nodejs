# nodejs


Using sendfile(2) with NodeJS
NodeJS provides an interface to using the sendfile(2) system call. Briefly, this system call allows the kernel to efficiently transport data from a file on-disk to a socket without round-tripping the data through user-space. This is one of the more important techniques that HTTP servers use to get good performance when serving static files.

Using this is slightly tricky in NodeJS, as the sendfile(2) call is not guaranteed to write all of a file's data to the given socket. Just like the write(2) system call, it can declare success after only writing a portion of the file contents to the given socket. This is commonly the case with non-blocking sockets, as files larger than the TCP send window cannot be buffered entirely in the TCP stack. At this point, one must wait until some of this outstanding data has been flushed to the other end of the TCP connection.

Without further ado, the following code implements a TCP server that uses sendfile(2) to transfer the contents of a file to every client that connects.
```js
var assert = require('assert');
var net = require('net');
var open = process.binding('fs').open;
var sendfile = process.binding('fs').sendfile;

if (process.argv.length < 4) {
    console.error('usage: sendfile <port> <path>');
    process.exit(1);
}

var port = parseInt(process.argv[2]);
var path = process.argv[3];
var bufSz = 1 << 10;

console.log('Sending ' + path + ' to all connections on port ' + port);

net.createServer(function(s) {
    open(path, process.O_RDONLY, 0, function(err, fd) {
        // Track our offset in the file outside of sendData() so that its value
        // is stable across multiple invocations
        var off = 0;

        var sendData = function() {
            // We only care about the 'drain' event if we're not done yet
            s.removeListener('drain', sendData);

            try {
                // Try to send file data until we either hit EOF, or fail the
                // write due to EAGAIN
                do {
                    nbytes = sendfile(s.fd, fd, off, bufSz);
                    off += nbytes;
                } while (nbytes > 0);

                s.end();
            } catch (e) {
                // Only EAGAIN is special; everything else is fatal
                if (e.errno !== process.EAGAIN) {
                    throw e;
                }

                // When the socket has room for more data, start pumping again
                s.on('drain', sendData);

                // Manually fire up the IOWatcher so that the 'drain' event
                // fires. The net.Stream class usually manages this for you,
                // but since we're going around it via sendfile(), we have to
                // do this manually.
                s._writeWatcher.start();
            }
        };

        if (err) {
            console.error(err);
            s.end();
            return;
        }

        // Kick off the transfer
        sendData();
    });
}).listen(port);

```
This is doing a couple of interesting things

We use the synchronous version of the sendfile API. We do this because we don't want to round-trip through the libeio thread pool.
We need to handle the EAGAIN error status from the sendfile(2) system call. NodeJS exposes this via a thrown exception. Rather than issuing another sendfile call right away, we wait until the socket is drained to try again (otherwise we're just busy-waiting). It's possible that the performance cost of generating and handling this exception is high enough that we'd be better off using the asynchronous version of the sendfile API.
We have to kick the write IOWatcher on the net.Stream instance ourselves to get the drain event to fire. This class only knows how to start the watcher itself when it notices a write(2) system call fail. Since we're using sendfile(2) behind its back, we have to tell it to do this explicitly.
We notice that we've hit the end of our source file when sendfile(2) returns 0 bytes written.
