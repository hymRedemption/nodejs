# Send File

In this lesson we will implement `res.sendfile` to allow the client to download a file from the server.

# Streaming Data

The [`Stream`](http://nodejs.org/api/stream.html) API provides the same abstract interface to do different kinds of IOs. You can read from a stream (a [Readable](http://nodejs.org/api/stream.html#stream_class_stream_readable)) or write to a stream (a [Writable](http://nodejs.org/api/stream.html#stream_class_stream_writable)), or both (a [Duplex](http://nodejs.org/api/stream.html#stream_class_stream_duplex)).

Some examples of stream in NodeJS are:

+ A TCP connection is a duplex stream.
+ A http request is a readable stream.
+ A file can be a readable or writable stream.
+ A unix pipe to a child process is a stream.
+ stdin and stdout are streams.

To print the content of a file to stdout:

```js
// create a Readable stream
file = fs.createReadStream("/foo.txt");

// write data to stdout
file.on("data",function(data) {
  process.stdout.write(data);
});
```

Or to send the content of a file to a TCP socket:

```js
var net = require("net");

file = fs.createReadStream("/foo.txt");

// send the content to localhost:4567
var socket = net.connect(4567,"localhost",function() {
  file.on("data",function(data) {
    socket.write(data);
  });
});
```

Because different kinds of streams all share the `write` and `read` methods, it's very easy to combine them for various purposes:

+ read file data and write to http response (client downloads a file)
+ read data from http request and write to a file (client uploads a file)
+ read data from a http request and write to a http response (http proxy)
+ read data from tcp/unix socket, and write to http response (reverse proxy)

# Fast Producer And Slow Consumer

Suppose your server is sending a 1 GB file to a slow tcp client:

+ The server reads the file at 100MB/s
+ The client connection is transferring at 10kb/s

This is not a problem for blocking IO. The server would read the file at ~10kb/s, because it is blocked by write:

```js
while(true) {
  var chunk = file.read(1024); // read 1024 bytes
  if(chunk == null)
    break;
  socket.write(chunk);         // write 1024 bytes, would block read
}
```

But because IO in NodeJS is non-blocking, the reader and writer are concurrent (they don't have to wait for each other). The server would read data from file as fast as possible, regardless of how fast the TCP socket can send data:

```
// file emits the "data" event as fast as possible
file.on("data",function(chunk) {
  // putting data in the socket's write buffer
  socket.write(chunk);
});
```

Because the operating system's TCP socket is only sending out data at 10kb/s, the chunks would have to be buffered in NodeJS's `socket` object. So after about 10 seconds the file would finish reading data, and the `socket` would have about 1GB worth of data buffered in the NodeJS process.

This is a classic case of the [producer–consumer problem](http://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem), where the producer sends data faster than the consumer can process.

There needs to be a way to synchronize IO streams.

The [pipe](http://nodejs.org/api/stream.html#stream_readable_pipe_destination_options) method for a Readable stream can copy data into Writable stream, while synchronizing the data transfer rate so the destination would not be overwhelmed.

For example, to read data from a file as fast as a socket can send data:

```js
// read at 10kb/s, send to socket at 10kb/s
file.pipe(socket);
```

# Backpressuring

How does `pipe` synchronize the read and write stream? It does so by backpressuring.

Let's see how we can synchronize the data transfer. We start with the non-synchronized version:

```js
// not synchronized.
file.on("data",function(chunk) {
  socket.write(chunk);
});
```

Backpressuring is a way for the writable stream to signal for the producer to stop. If `writable.write(chunk)` returns `false`, then we should [`pause`](http://nodejs.org/api/stream.html#stream_readable_pause) the data flow.

```js
file.on("data",function(chunk) {
  var ok = socket.write(chunk);
  if(ok == false) {
    // stop reading data
    file.pause();
  }
});
```

When `socket.write(chunk)` returns false, it's indicating that `chunk` is queued in a buffer in nodejs, waiting to be sent out to the underlying system socket. You can keep writing to the nodejs socket, but it will grow the nodejs buffer.

After the socket object's buffer had sent buffered data to the tcp socket, we can resume the data flow. By listening to the `drain` event on the writable stream, we'd get notified when the writable stream is ready for write again:

```js
file.on("data",function(chunk) {
  var ok = socket.write(chunk);
  if(ok == false) {
    // stop reading data
    file.pause();

    // wait for the tcp socket to finish writing
    socket.once("drain",function() {
      // resume read
      file.resume();
    });
  }
});
```

In summary, to stop down data flow:

+ `writable.write` returns false to signal backpressure
+ `readable.pause` to stop reading data
+ wait for writable's buffer to drain

And to resume data flow:

+ `writable.once("drain")` signals that it's ready for write
+ `readable.resume` to continue reading data

This is essentially what `pipe` does to synchronize readable and writable streams.

# Implement Basic res.sendfile

**Implement**: `res.stream(stream)` should stream data to client


Pass: `mocha verify/sendfile_spec.js -R spec -g "stream data"`

```
  Basic res.sendfile
    stream data:
      ✓ can stream data to client
      ✓ returns empty body for head
```
Pass: `mocha verify/sendfile_spec.js -R spec -g 'can stream data to client'`

**Implement**: `res.sendfile(data,options)` to stream file data to client

Example:

```js
// send file relative to a given root
res.sendfile("/data.txt",{root: process.cwd()})

// send file using given path
res.sendfile("/var/www/robot.txt")
```

Pass: `mocha verify/sendfile_spec.js -R spec -g 'stream file data'`

```
Basic res.sendfile
  stream file data:
    ✓ reads file from path
    ✓ reads file from path relative to root
```

Hint: use `path.normalize` to get rid of double slashes in paths like `/a//b/c`

**Implement**: Set Content-Type and Content-Length headers

Hint: use [`fs.stat`](http://nodejs.org/api/fs.html#fs_class_fs_stats) to read file metadata and get the file size.

Pass: `mocha verify/sendfile_spec.js -R spec -g 'content headers'`

```
Basic res.sendfile
  content headers:
    ✓ sets content type
    ✓ sets content length
```

**Implement**: respond with error for bad paths

+ 404 if path is not found
+ 403 if path is a directory
+ 403 if path contains ".." in it

Pass: `mocha verify/sendfile_spec.js -R spec -g 'path checking'`

```
Basic res.sendfile
  path checking:
    ✓ should 404 if fs.stat fails
    ✓ should 403 if file is a directory
    ✓ should 403 if path contains ..
```

# Range Request

Normally if a connection breaks, the client would have to restart downloading a file from the beginning. But if the client knows that the partially downloaded file is 1000bytes so far, it can continue to download from the 1000th byte by using the `Range` header.

Let's see how this works if we were downloading the Ubuntu ISO image:

```
> curl -I  http://releases.ubuntu.com/12.04.4/ubuntu-12.04.4-desktop-amd64.iso
HTTP/1.1 200 OK
Date: Thu, 17 Apr 2014 16:05:09 GMT
Server: Apache/2.2.22 (Ubuntu)
Last-Modified: Tue, 04 Feb 2014 12:11:03 GMT
ETag: "c00ef-2dd00000-4f19388b6a3c0"
Accept-Ranges: bytes
Content-Length: 768606208
Content-Type: application/x-iso9660-image
```

The returned header `Accept-Ranges: bytes` indicates that the server supports Range request for this url. Not every url would support Range.

We can get the first 128 bytes by including the `Range` header in the request:

```
curl -s -H "Range: bytes=0-127"  http://releases.ubuntu.com/12.04.4/ubuntu-12.04.4-desktop-amd64.iso
E��3�ռ|��f11SfQW�ݎ�|��KR�A��U1��r��U�u�t
                                        f�BZQ�[Q�P
```

The output looks messed up because the file has unprintable chars. We can use `hexdump` to format the raw bytes in hexadecimals:

```
> curl -s -H "Range: bytes=0-127"  http://releases.ubuntu.com/12.04.4/ubuntu-12.04.4-desktop-amd64.iso | hexdump -C
00000000  45 52 08 00 00 00 90 90  00 00 00 00 00 00 00 00  |ER..............|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000020  33 ed fa 8e d5 bc 00 7c  fb fc 66 31 db 66 31 c9  |3......|..f1.f1.|
00000030  66 53 66 51 06 57 8e dd  8e c5 52 be 00 7c bf 00  |fSfQ.W....R..|..|
00000040  06 b9 00 01 f3 a5 ea 4b  06 00 00 52 b4 41 bb aa  |.......K...R.A..|
00000050  55 31 c9 30 f6 f9 cd 13  72 16 81 fb 55 aa 75 10  |U1.0....r...U.u.|
00000060  83 e1 01 74 0b 66 c7 06  f1 06 b4 42 eb 15 eb 00  |...t.f.....B....|
00000070  5a 51 b4 08 cd 13 83 e1  3f 5b 51 0f b6 c6 40 50  |ZQ......?[Q...@P|
```

In this dump we can see 128 bytes in total. Each byte is a hex number like "45", "8e", etc. Each row has 16 bytes.

And we can get the second 128 bytes:

```
curl -s -H "Range: bytes=128-255"  http://releases.ubuntu.com/12.04.4/ubuntu-12.04.4-desktop-amd64.iso | hexdump -C
00000000  f7 e1 53 52 50 bb 00 7c  b9 04 00 66 a1 b0 07 e8  |..SRP..|...f....|
00000010  44 00 0f 82 80 00 66 40  80 c7 02 e2 f2 66 81 3e  |D.....f@.....f.>|
00000020  40 7c fb c0 78 70 75 09  fa bc ec 7b ea 44 7c 00  |@|..xpu....{.D|.|
00000030  00 e8 83 00 69 73 6f 6c  69 6e 75 78 2e 62 69 6e  |....isolinux.bin|
00000040  20 6d 69 73 73 69 6e 67  20 6f 72 20 63 6f 72 72  | missing or corr|
00000050  75 70 74 2e 0d 0a 66 60  66 31 d2 66 03 06 f8 7b  |upt...f`f1.f...{|
00000060  66 13 16 fc 7b 66 52 66  50 06 53 6a 01 6a 10 89  |f...{fRfP.Sj.j..|
00000070  e6 66 f7 36 e8 7b c0 e4  06 88 e1 88 c5 92 f6 36  |.f.6.{.........6|
```

A successful range request should get a 206 response:

```
curl -s -I -H "Range: bytes=128-255"  http://releases.ubuntu.com/12.04.4/ubuntu-12.04.4-desktop-amd64.iso
HTTP/1.1 206 Partial Content
Date: Thu, 17 Apr 2014 16:15:02 GMT
Server: Apache/2.2.22 (Ubuntu)
Last-Modified: Tue, 04 Feb 2014 12:11:03 GMT
ETag: "c00ef-2dd00000-4f19388b6a3c0"
Accept-Ranges: bytes
Content-Length: 128
Content-Range: bytes 128-255/768606208
Content-Type: application/x-iso9660-image
```

+ The `Content-Length` is the number of bytes returned for this range request.
+ `Content-Range` has two parts:
  + `128-255` is the range requested.
  + `768606208` is the total number of bytes for this resource.
+ `ETag` is the same value as the full file.

For another example, see how [Chrome retrieves an MP4 video](http://stackoverflow.com/questions/8293687/sample-http-range-request-session)

# Implement Range Request Support

We'll use the `range-parser` package to help parse the `Range` header:

```
npm install range-parser@1.0.0 --save
```

Some examples of using range-parser

```js
// first 20 bytes
> rparser = require("range-parser")
> r = rparser(200, 'bytes=0-19')
[ { start: 0, end: 19 },
  type: 'bytes' ]

// get the end 30 bytes
> r = rparser(200, 'bytes=-30')
[ { start: 170, end: 199 },
  type: 'bytes' ]

// get bytes from 30 to the end
> r = rparser(200, 'bytes=29-')
[ { start: 29, end: 199 },
  type: 'bytes' ]

// unsatisfiable range
> r = rparser(200, 'bytes=20-10')
-1

// invalid range
> r = rparser(200, 'invalid range')
-2

// we'll ignore this case..
> r = rparser(200, 'bytes=0-10,20-30')
[ { start: 0, end: 10 },
  { start: 20, end: 30 },
  type: 'bytes' ]
```
**Implement**: range support

Hint: The options for `fs.createReadStream(path, options)` can include `start` and `end` values to read a range of bytes from the file instead of the entire file.

Pass: `mocha verify/sendfile_spec.js -R spec -g 'Range support'`

```
Basic res.sendfile
  Range support
    ✓ sets Accept-Range
    ✓ returns 206 for Range get
    ✓ returns 416 for unsatisfiable range
    ✓ ignores Range if it is invalid
```