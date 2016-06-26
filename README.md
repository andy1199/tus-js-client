# tus-js-client [![Build Status](https://travis-ci.org/tus/tus-js-client.svg?branch=master)](https://travis-ci.org/tus/tus-js-client)
A pure JavaScript client for the [tus resumable upload protocol](http://tus.io)
which works in browser environments and Node.js.

## Example

```js
input.addEventListener("change", function(e) {
    // Get the selected file from the input element
    var file = e.target.files[0]

    // Create a new tus upload
    var upload = new tus.Upload(file, {
        endpoint: "http://localhost:1080/files/",
        onError: function(error) {
            console.log("Failed because: " + error)
        },
        onProgress: function(bytesUploaded, bytesTotal) {
            var percentage = (bytesUploaded / bytesTotal * 100).toFixed(2)
            console.log(bytesUploaded, bytesTotal, percentage + "%")
        },
        onSuccess: function() {
            console.log("Download %s from %s", upload.file.name, upload.url)
        }
    })

    // Start the upload
    upload.start()
})
```

## Installation

The sources are compiled into a [UMD](https://github.com/umdjs/umd)
(`dist/tus.js`) which can be loaded using different approaches:

* **Embed using a script tag:** `<script src="dist/tus.js"></script>` and access
the tus methods using the `tus` property in `window`
* **Install from NPM:** `npm install tus-js-client`:
`var tus = require("tus-js-client")`
* **Define using AMD:** `define("alpha", ["dist/tus.js"], function(tus) {})`

## Browser support

[![Sauce Test Status](https://saucelabs.com/browser-matrix/marius_transloadit.svg)](https://saucelabs.com/u/marius_transloadit)

We use localStorage, XMLHttpRequest2, the File API and Blob API. About 85% of
today's users should be able to use this software, calculated using
[iwanttouse.com](http://www.iwanttouse.com/#namevalue-storage,xhr2,fileapi,blobbuilder).

## Node compatibility

Since Node's environment is quite different than a browser's runtime and
provides other capabilities but also restrictions, tus-js-client will have a
slightly changed behavior when used in the context of a Node.js application:

* The `onProgress` event handler will not be invoked during a request's body
is transmitted but only once a `PATCH` request is finished. Therefore, it may
only be emitted once, depending on the value of the `chunkSize` value. Basically,
it behaves very similar to the `onChunkComplete` event.

* As the Web Storage API is only available in browser environments,
tus-js-client will not be able store the URLs of created uploads allowing
automatic resuming. Please consult the documentation for the `tus.canStoreURLs`
for more information on this specific topic.

* The `tus.Upload` constructor will only accept instances of `buffer.Buffer`
and `stream.Readable` as file inputs. If you are passing a readable stream as
this argument, you must set the `chunkSize` option to a finite integer value
because the chunk, which is currently being uploaded, will be held in memory
allowing automatic retries, e.g. after connection interruptions. Therefore
additional care should be taken when choosing the appropriate value for your
specific application to control memory consumption.

* If you call the `tus.Upload` constructor with an instance of the
`fs.ReadStream`, the above point does not apply, meaning *no* chunk will be held
in memory. Instead, tus-js-client will create it's own stream starting at the
needed position using `fs.createReadStream`. If you want to disable this
functionality, you may want to wrap the `fs.ReadStream` into a
`stream.PassThrough`.

Finally, you may be interested in the `demo/node.js` example which demonstrates
a simple example on how to easily use tus-js-client using Node.js.

## Internals

Once a new file should be uploaded the client will create a new upload resource
on the server using a `POST` request. A successful response will contain a
`Location` header pointing to the upload URL. This URL will be used to transfer
the file to the server using one or multiple `PATCH` requests.
In addition tus-js-client will generate a unique fingerprint for every file and
store it and the upload URL using the Web Storage API. If the upload is
interrupted or aborted manually, the client is able to resume the upload by
retrieving the upload URL using the fingerprint. The client is even able to
resume after you close your browser or shut down your device. Now the client can
continue to send `PATCH` requests to the server until the upload is finished.

## Extension support

The tus specification defines multiple [extensions](http://tus.io/protocols/resumable-upload.html#protocol-extensions) which can be optionally
implemented beside the core protocol enabling specific functionality. Not all
of these extensions are interesting or even useful for a client-side library
and therefore support for all of them in tus-js-client is not guaranteed.

* The **Creation** extension is mostly implemented and is used for creating the
upload. Deferring the upload's length is not possible at the moment.

* The Checksum extension requires that the checksum is calculated inside the
browser. While this is totally doable today, it's particularly expensive and
time intensive for bigger files and on mobile devices. One solution is to
utilize the new Web Crypto API, which probably offers better performance and
security, but you could argue whether it has reached critical mass yet.

* The Concatenation extension is mostly meant for parallel uploads where you
need to utilize multiple HTTP connections. In most cases, this does not apply
to the environment of the browser but it can also be used for different things.

At the moment, coverage for these extensions is not great but we promise to
improve this situation in the near future.

## Documentation

### tus.isSupported

A boolean indicating whether the current browser has the features necessary to
use tus-js-client. This can be used to test support and warn the user.

### tus.canStoreURLs

A boolean indicating whether the current environment allows storing URLs
enabling the corresponding upload to be resumed if the same file (identified
using fingerprinting) is passed to the constructor again. Since this storage
mechanism is currently bound to the Web Storage API, this value will only yield
to `true` if we are in a browser environment which provides access to the
`localStorage` object. Please be aware that in some cases, e.g. a sandboxed
iframe, the Web Storage API is provided but cannot be used without causing
security errors. In these special situations or if no Web Storage API is
available, `canStoreURLs` is set to `false`.

### tus.defaultOptions

A object containing the default options used when creating a new upload:

* `endpoint = ""`: a URL which will be used to create new uploads
* `fingerprint`: a function used to generate a unique string from a
corresponding File object. This used to store the URL for an upload to resume.
This option is only used if the `resume` flag is set to true.
* `resume = true`: a boolean indicating whether the client should attempt to
resume the upload if the upload has been started in the past. This includes
storing the file's fingerprint. Use `false` to force an entire reupload.
* `onProgress = null`: a function that will be called each time progress
information is available. The arguments will be `bytesSent` and `bytesTotal`.
* `onChunkComplete = null`: a function that will be called each time a chunk
has been successfully uploaded. The arguments will be `chunkSize`,
`bytesAccepted`, `bytesTotal`.
* `onSuccess = null`: a function called when the upload finished successfully.
* `onError = null`: a function called once an error appears. The arguments will
be an Error instance.
* `headers = {}`: an object with custom header values used in all requests.
* `withCredentials = false`: a boolean which is be used as the value for
`withCredentials` in all XMLHttpRequests to use Cookies in requests. The
remote server must accept CORS and credentials.
* `chunkSize = Infinity`: a number indicating the maximum size of a chunk
uploaded in a single request
* `metadata = {}`: an object with string values used as additional meta data
which will be passed along to the server when (and only when) creating a new
upload. Can be used for filenames, file types etc.
* `uploadUrl = null`: a URL which will be used to directly attempt a resume
without generating the fingerprint and looking it up before. If this attempt
fails it will fall back to creating a new upload using the URL specified in
`endpoint`. This will also force an attempt even if resuming has been disabled
by setting `resume` to `false`.
* `uploadSize = null`: an integer representing the size of the file in bytes.
This will only be used if the size cannot be automatically calculated. This
is currently only used and required if you supply a `Readable` stream as the
file to upload. You may also use this to limit the position until which a file
will be uploaded.

### new tus.Upload(file, options)

Create a new tus.Upload object. The upload will not be started automatically,
use `start` to do so.

The `file` argument must be an instance of `File` or `Blob` if you are in a
browser environment. If it is executed using Node.js, the allowed types are
`Buffer` and a `Readable` stream.
The `options` argument will be merged deeply with `tus.defaultOptions`.

### tus.Upload#options

The `options` argument used in the constructor merged deeply with
`tus.defaultOptions`.

### tus.Upload#file

The `file` argument used in the constructor.

### tus.Upload#url

The URL used to upload the file. Leave it set to `null` and the client will
create a new upload and set this property to the new upload URL.
You may supply your own URL using this property and the client will try to
resume using this URL.

### tus.Upload#start()

Start or resume the upload using the specified file. If no `file` property is
available the error handler will be called. If you supplied your own URL using
the `url` property the client will try to resume using this URL.
If not, the client will look up if the file has been (fully or partially)
uploaded and tries to resume.
If no upload can be resume it will create a new upload using the supplied
`endpoint` option.

### tus.Upload#abort()

Abort the currently running upload request and don't continue. You can resume
the upload by calling the `start` method again.

### Difference between onProgress and onChunkComplete

When configuring a new uploader, the `onProgress` and `onChunkComplete`
callbacks are available. While they may seem to be equal based on their
naming and the arguments, they provide different information in reality.
Progress events are emitted using the `onProgress` option and provide numbers
about how must data has been sent to the server. However, this data may not
have been received or accepted by the endpoint. Imagine a network outage where
the browser reports to have successfully sent 100 bytes, but none of them ever
reach the backend. In order to provide reliable information about whether the
chunks have been accepted by the server, `onChunkComplete` is only invoked if
we have evidence that the remote endpoint has received and accepted the
uploaded bytes. When consuming this functionality, the `chunkSize` option is
from high importance since the callback will and invoked if an entire chunk
has been uploaded.

## Building

```bash
# Install dependencies
npm install

# Build dist/tus.js a single time
npm run build

# Build all dist files (including minification)
npm run dist

# Watch source and rebuild
npm run watch
```

## Testing

Tests are implemented using Jasmine and can be found in the `test/` directory.
In order to run the tests, open `test/SpecRunner.html` in a browser and you
should see a visual representation of the test results. No web server is
required, you can open `SpecRunner.html` using the `file:///` protocol.

In addition, if do not want to download the repository, you can run the latest
tests online by visiting [https://rawgit.com/tus/tus-js-client/master/test/SpecRunner.html](https://rawgit.com/tus/tus-js-client/master/test/SpecRunner.html).

Tests can also be run on SauceLabs' cloud infrastructure using `npm test`.
Before using this command, you have to set up your SauceLabs account by filling
the `SAUCE_USERNAME` and `SAUCE_ACCESS_KEY` variables else the command will fail.

## License

MIT
