# High Level Amazon S3 Client

## Features

 * Automatically retry a configurable number of times when S3 returns an error.
 * Includes logic to make multiple requests when there is a 1000 object limit.
 * Ability to set a limit on the maximum parallelization of S3 requests.
   Retries get pushed to the end of the paralellization queue.
 * Ability to sync a dir to and from S3.
 * Progress reporting.
 * Supports files of any size (up to S3's maximum 5 TB object size limit).
 * Uploads large files quickly using parallel multipart uploads.
 * Uses heuristics to compute multipart ETags client-side to avoid uploading
   or downloading files unnecessarily.
 * Automatically provide Content-Type for uploads based on file extension.
 * Supports uploading a stream with unknown Content-Length. Streams
   simultaneously to S3 as well as a temp file, to keep memory footprint low
   and be able to retry when S3 fails.

See also the companion CLI tool which is meant to be a drop-in replacement for
s3cmd: [s3-cli](https://github.com/andrewrk/node-s3-cli).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](http://doctoc.herokuapp.com/)*

- [Synopsis](#synopsis)
  - [Create a client](#create-a-client)
  - [Create a client from existing AWS.S3 object](#create-a-client-from-existing-awss3-object)
  - [Upload a file to S3](#upload-a-file-to-s3)
  - [Download a file from S3](#download-a-file-from-s3)
  - [Sync a directory to S3](#sync-a-directory-to-s3)
  - [Upload a stream to S3](#upload-a-stream-to-s3)
- [Tips](#tips)
- [API Documentation](#api-documentation)
  - [s3.createClient(options)](#s3createclientoptions)
  - [s3.getPublicUrl(bucket, key, [bucketLocation])](#s3getpublicurlbucket-key-bucketlocation)
  - [s3.getPublicUrlHttp(bucket, key)](#s3getpublicurlhttpbucket-key)
  - [client.uploadDir(params)](#clientuploaddirparams)
  - [client.uploadStream(params)](#clientuploadstreamparams)
  - [client.uploadFile(params)](#clientuploadfileparams)
  - [client.uploadFd(params)](#clientuploadfdparams)
  - [client.downloadDir(params)](#clientdownloaddirparams)
  - [client.downloadFile(params)](#clientdownloadfileparams)
  - [client.listObjects(params)](#clientlistobjectsparams)
  - [client.deleteObjects(s3Params)](#clientdeleteobjectss3params)
  - [client.deleteDir(s3Params)](#clientdeletedirs3params)
  - [client.copyObject(s3Params)](#clientcopyobjects3params)
  - [client.moveObject(s3Params)](#clientmoveobjects3params)
- [Testing](#testing)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Synopsis

### Create a client

```js
var s3 = require('s3');

var client = s3.createClient({
  maxAsyncS3: 20,     // this is the default
  s3RetryCount: 3,    // this is the default
  s3RetryDelay: 1000, // this is the default
  multipartUploadThreshold: 20971520, // this is the default (20 MB)
  multipartUploadSize: 15728640, // this is the default (15 MB)
  s3Options: {
    accessKeyId: "your s3 key",
    secretAccessKey: "your s3 secret",
    // any other options are passed to new AWS.S3()
    // See: http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Config.html#constructor-property
  },
});
```

### Create a client from existing AWS.S3 object

```js
var s3 = require('s3');
var awsS3Client = new AWS.S3(s3Options);
var options = {
  s3Client: awsS3Client,
};
var client = s3.fromAwsSdkS3(options);
```

### Upload a file to S3

```js
var params = {
  localFile: "some/local/file",

  s3Params: {
    Bucket: "s3 bucket name",
    Key: "some/remote/file",
    // other options supported by putObject, except Body and ContentLength.
    // See: http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property
  },
};
var uploader = client.uploadFile(params);
uploader.on('error', function(err) {
  console.error("unable to upload:", err.stack);
});
uploader.on('progress', function() {
  console.log("progress", uploader.progressMd5Amount,
            uploader.progressAmount, uploader.progressTotal);
});
uploader.on('end', function() {
  console.log("done uploading");
});
```

### Download a file from S3

```js
var params = {
  localFile: "some/local/file",

  s3Params: {
    Bucket: "s3 bucket name",
    Key: "some/remote/file",
    // other options supported by getObject
    // See: http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getObject-property
  },
};
var downloader = client.downloadFile(params);
downloader.on('error', function(err) {
  console.error("unable to download:", err.stack);
});
downloader.on('progress', function() {
  console.log("progress", downloader.progressAmount, downloader.progressTotal);
});
downloader.on('end', function() {
  console.log("done downloading");
});
```

### Sync a directory to S3

```js
var params = {
  localDir: "some/local/dir",
  deleteRemoved: true, // default false, whether to remove s3 objects
                       // that have no corresponding local file.

  s3Params: {
    Bucket: "s3 bucket name",
    Prefix: "some/remote/dir/",
    // other options supported by putObject, except Body and ContentLength.
    // See: http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property
  },
};
var uploader = client.uploadDir(params);
uploader.on('error', function(err) {
  console.error("unable to sync:", err.stack);
});
uploader.on('progress', function() {
  console.log("progress", uploader.progressAmount, uploader.progressTotal);
});
uploader.on('end', function() {
  console.log("done uploading");
});
```

### Upload a stream to S3

```js
var params = {
  stream: someReadableStream,
  filename: "filename.txt", // optional, provide a filename so that
                            // Content-Type can be automatically populated.
  s3Params: {
    Bucket: "s3 bucket name",
    Key: "some/remote/file",
    ContentLength: 1024,  // optional, some optimizations can be made if you
                          // know the Content-Length ahead of time
    // other options supported by putObject, except Body
    // See: http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property
  },
};
var uploader = client.uploadStream(params);
uploader.on('error', function(err) {
  console.error("unable to upload:", err.stack);
});
uploader.on('progress', function() {
  console.log("progress", uploader.progressAmount, uploader.progressTotal);
});
uploader.on('end', function() {
  console.log("done uploading");
});
```

## Tips

 * Consider increasing the socket pool size in the `http` and `https` global
   agents. This will improve bandwidth when using `uploadDir` and `downloadDir`
   functions. For example:

   ```js
   http.globalAgent.maxSockets = https.globalAgent.maxSockets = 20;
   ```

## API Documentation

### s3.createClient(options)

Creates an S3 client.

`options`:

 * `s3Client` - optional, an instance of `AWS.S3`. Leave blank if you provide `s3Options`.
 * `s3Options` - optional. leave blank if you provide `s3Client`.
   - See AWS SDK documentation for available options which are passed to `new AWS.S3()`:
     http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Config.html#constructor-property
 * `maxAsyncS3` - maximum number of simultaneous requests this client will
   ever have open to S3. defaults to `20`.
 * `s3RetryCount` - how many times to try an S3 operation before giving up.
   Default 3.
 * `s3RetryDelay` - how many milliseconds to wait before retrying an S3
   operation. Default 1000.
 * `multipartUploadThreshold` - if a file is this many bytes or greater, it
   will be uploaded via a multipart request. Default is 20MB. Minimum is 5MB.
   Maximum is 5GB.
 * `multipartUploadSize` - when uploading via multipart, this is the part size.
   The minimum size is 5MB. The maximum size is 5GB. Default is 15MB. Note that
   S3 has a maximum of 10000 parts for a multipart upload, so if this value is
   too small, it will be ignored in favor of the minimum necessary value
   required to upload the file.

### s3.getPublicUrl(bucket, key, [bucketLocation])

 * `bucket` S3 bucket
 * `key` S3 key
 * `bucketLocation` string, one of these:
   - "" (default) - US Standard
   - "eu-west-1"
   - "us-west-1"
   - "us-west-2"
   - "ap-southeast-1"
   - "ap-southeast-2"
   - "ap-northeast-1"
   - "sa-east-1"

You can find out your bucket location programatically by using this API:
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getBucketLocation-property

returns a string which looks like this:

`https://s3.amazonaws.com/bucket/key`

or maybe this if you are not in US Standard:

`https://s3-eu-west-1.amazonaws.com/bucket/key`

### s3.getPublicUrlHttp(bucket, key)

 * `bucket` S3 Bucket
 * `key` S3 Key

Works for any region, and returns a string which looks like this:

`http://bucket.s3.amazonaws.com/key`

### client.uploadDir(params)

Syncs an entire directory to S3.

`params`:

 * `localDir` - source path on local file system to sync to S3
 * `s3Params`
   - `Prefix` (required)
   - `Bucket` (required)
 * (optional) `deleteRemoved` - delete s3 objects with no corresponding local file.
   default false
 * (optional) `getS3Params` - function which will be called for every file that
   needs to be uploaded. See below.
 * (optional) `defaultContentType`: Unless you explicitly set the `ContentType`
   parameter in `s3Params`, it will be automatically set for you based on the
   file extension of `localFile`. If the extension is unrecognized,
   `defaultContentType` will be used instead. Defaults to
   `application/octet-stream`.

```js
function getS3Params(localFile, stat, callback) {
  // call callback like this:
  var err = new Error(...); // only if there is an error
  var s3Params = { // if there is no error
    ContentType: getMimeType(localFile), // just an example
  };
  // pass `null` for `s3Params` if you want to skip uploading this file.
  callback(err, s3Params);
}
```

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`
 * `progressMd5Amount`
 * `progressMd5Total`
 * `deleteAmount`
 * `deleteTotal`
 * `filesFound`
 * `objectsFound`
 * `doneFindingFiles`
 * `doneFindingObjects`
 * `doneMd5`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when all files are uploaded
 * `'progress'` - emitted when any of the above progress properties change.

`uploadDir` works like this:

 0. Start listing all S3 objects for the target `Prefix`. S3 guarantees
    returned objects to be in sorted order.
 0. Meanwhile, recursively find all files in `localDir`.
 0. Once all local files are found, we sort them (the same way that S3 sorts).
 0. Next we iterate over the sorted local file list one at a time, computing
    MD5 sums.
 0. Now S3 object listing and MD5 sum computing are happening in parallel. As
    each operation progresses we compare both sorted lists side-by-side,
    iterating over them one at a time, uploading files whose MD5 sums don't
    match the remote object (or the remote object is missing), and, if
    `deleteRemoved` is set, deleting remote objects whose corresponding local
    files are missing.

### client.uploadStream(params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property

`params`:

 * `s3Params`: params to pass to AWS SDK `putObject`, except for `Body`.
   Some optimizations can be made if you provide `ContentLength`, but it is
   optional.
 * (optional) `filename`: provide a filename so that Content-Type can be
   automatically populated.
 * (optional) `partSize`: This is the size of each part in the multipart
   upload. Defaults to 5MB which is the minimum S3 allows. Note that S3 has a
   maximum of 10000 parts, so `10000 * partSize` is the maximum size of the
   stream that you can upload. The maximum value for `partSize` is 5GB.
 * (optional) `defaultContentType`: Unless you explicitly set the `ContentType`
   parameter in `s3Params`, it will be automatically set for you based on the
   file extension of `filename`. If the extension is unrecognized,
   `defaultContentType` will be used instead. Defaults to
   `application/octet-stream`.

The difference between using AWS SDK `putObject` and this one:

 * This works with a stream, not a file or buffer.
 * Does not require knowing `ContentLength` ahead of time.
 * Performs a multipart upload.
 * If the reported MD5 upon upload completion does not match, it retries.
 * Retry based on the client's retry settings. Writes the stream to a temp
   file in case retry is needed. Temp file is automatically cleaned up.
 * Progress reporting.
 * Sets the `ContentType` based on file extension if you do not provide it.

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end' (data)` - emitted when the file is uploaded successfully
   - `data` is the same object that you get from `putObject` in AWS SDK
 * `'progress'` - emitted when `progressAmount` or `progressTotal` properties
   change. Note that it is possible for progress to go backwards when S3 fails
   and the upload must be retried.
 * `'fileOpened' (fdSlicer)` - emitted when `localFile` has been opened. The file
   is opened with the [fd-slicer](https://github.com/andrewrk/node-fd-slicer)
   module because we might need to read from multiple locations in the file at
   the same time. `fdSlicer` is an object for which you can call
   `createReadStream(options)`. See the fd-slicer README for more information.

And these methods:

 * `abort()` - call this to stop the operation.

`uploadStream` works like this:

 0. Open a temp file with w+ flags and start streaming to it.
 0. If `ContentLength` is provided, we can proceed. Otherwise wait until
    `partSize` bytes have been written to the temp file or the stream is
    complete.
 0. If the stream is complete, begin a `putObject`. Otherwise begin a multipart
    upload.
 0. Stream the still-open temp file into the upload. Streaming from disk rather
    than buffering `partSize` in memory gives us a low memory footprint.
 0. If the upload completes successfully to S3, then we remove the temp file
    and we're done.
 0. If S3 fails, then we retry the upload using the temp file (we still haven't
    closed it). The retry will know the Content-Length. If all the retries fail
    or we succeed then we remove the temp file.

### client.uploadFile(params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property

`params`:

 * `s3Params`: params to pass to AWS SDK `putObject`.
 * `localFile`: path to the file on disk you want to upload to S3.
 * (optional) `defaultContentType`: Unless you explicitly set the `ContentType`
   parameter in `s3Params`, it will be automatically set for you based on the
   file extension of `localFile`. If the extension is unrecognized,
   `defaultContentType` will be used instead. Defaults to
   `application/octet-stream`.

The difference between using AWS SDK `putObject` and this one:

 * This works with files, not streams or buffers.
 * If the reported MD5 upon upload completion does not match, it retries.
 * If the file size is large enough, uses multipart upload to upload parts in
   parallel.
 * Retry based on the client's retry settings.
 * Progress reporting.
 * Sets the `ContentType` based on file extension if you do not provide it.

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end' (data)` - emitted when the file is uploaded successfully
   - `data` is the same object that you get from `putObject` in AWS SDK
 * `'progress'` - emitted when `progressAmount` or `progressTotal` properties
   change. Note that it is possible for progress to go backwards when S3 fails
   and the upload must be retried.
 * `'fileOpened' (fdSlicer)` - emitted when `localFile` has been opened. The file
   is opened with the [fd-slicer](https://github.com/andrewrk/node-fd-slicer)
   module because we might need to read from multiple locations in the file at
   the same time. `fdSlicer` is an object for which you can call
   `createReadStream(options)`. See the fd-slicer README for more information.

And these methods:

 * `abort()` - call this to stop the operation.

### client.uploadFd(params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property

`params`:

 * `fd`: The file descriptor to read from.
 * `s3Params`: params to pass to AWS SDK `putObject`, except for `Body` and
   `ContentLength`.
 * (optional) `filename`: provide a filename so that Content-Type can be
   automatically populated.
 * (optional) `defaultContentType`: Unless you explicitly set the `ContentType`
   parameter in `s3Params`, it will be automatically set for you based on the
   file extension of `filename`. If the extension is unrecognized,
   `defaultContentType` will be used instead. Defaults to
   `application/octet-stream`.

The difference between using AWS SDK `putObject` and this one:

 * This works with fds, not files, streams, or buffers.
 * If the reported MD5 upon upload completion does not match, it retries.
 * If the file size is large enough, uses multipart upload to upload parts in
   parallel.
 * Retry based on the client's retry settings.
 * Progress reporting.
 * Sets the `ContentType` based on file extension if you do not provide it (as
   long as you provide the `filename` parameter).

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end' (data)` - emitted when the file is uploaded successfully
   - `data` is the same object that you get from `putObject` in AWS SDK
 * `'progress'` - emitted when `progressAmount` or `progressTotal` properties
   change. Note that it is possible for progress to go backwards when S3 fails
   and the upload must be retried.
 * `'fileOpened' (fdSlicer)` - emitted when `localFile` has been opened. The file
   is opened with the [fd-slicer](https://github.com/andrewrk/node-fd-slicer)
   module because we might need to read from multiple locations in the file at
   the same time. `fdSlicer` is an object for which you can call
   `createReadStream(options)`. See the fd-slicer README for more information.

And these methods:

 * `abort()` - call this to stop the operation.

### client.downloadDir(params)

Syncs an entire directory from S3.

`params`:

 * `localDir` - destination directory on local file system to sync to
 * `s3Params`
   - `Prefix` (required)
   - `Bucket` (required)
 * (optional) `deleteRemoved` - delete local files with no corresponding s3 object. default `false`
 * (optional) `getS3Params` - function which will be called for every object that
   needs to be downloaded. See below.

```js
function getS3Params(localFile, s3Object, callback) {
  // localFile is the destination path where the object will be written to
  // s3Object is same as one element in the `Contents` array from here:
  // http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#listObjects-property

  // call callback like this:
  var err = new Error(...); // only if there is an error
  var s3Params = { // if there is no error
    VersionId: "abcd", // just an example
  };
  // pass `null` for `s3Params` if you want to skip downloading this object.
  callback(err, s3Params);
}
```

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`
 * `progressMd5Amount`
 * `progressMd5Total`
 * `deleteAmount`
 * `deleteTotal`
 * `filesFound`
 * `objectsFound`
 * `doneFindingFiles`
 * `doneFindingObjects`
 * `doneMd5`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when all files are uploaded
 * `'progress'` - emitted when any of the progress properties above change

`downloadDir` works like this:

 0. Start listing all S3 objects for the target `Prefix`. S3 guarantees
    returned objects to be in sorted order.
 0. Meanwhile, recursively find all files in `localDir`.
 0. Once all local files are found, we sort them (the same way that S3 sorts).
 0. Next we iterate over the sorted local file list one at a time, computing
    MD5 sums.
 0. Now S3 object listing and MD5 sum computing are happening in parallel. As
    each operation progresses we compare both sorted lists side-by-side,
    iterating over them one at a time, downloading objects whose MD5 sums don't
    match the local file (or the local file is missing), and, if
    `deleteRemoved` is set, deleting local files whose corresponding objects
    are missing.

### client.downloadFile(params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getObject-property

`params`:

 * `localFile` - the destination path on disk to write the s3 object into
 * `s3Params`: params to pass to AWS SDK `getObject`.

The difference between using AWS SDK `getObject` and this one:

 * This works with a destination file, not a stream or a buffer.
 * If the reported MD5 upon download completion does not match, it retries.
 * Retry based on the client's retry settings.
 * Progress reporting.

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when the file is uploaded successfully
 * `'progress'` - emitted when `progressAmount` and `progressTotal`
   properties change.

### client.listObjects(params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#listObjects-property

`params`:

 * `s3Params` - params to pass to AWS SDK `listObjects`.
 * (optional) `recursive` - `true` or `false` whether or not you want to recurse
   into directories. Default `false`.

Note that if you set `Delimiter` in `s3Params` then you will get a list of
objects and folders in the directory you specify. You probably do not want to
set `recursive` to `true` at the same time as specifying a `Delimiter` because
this will cause a request per directory. If you want all objects that share a
prefix, leave the `Delimiter` option `null` or `undefined`.

Be sure that `s3Params.Prefix` ends with a trailing slash (`/`) unless you
are requesting the top-level listing, in which case `s3Params.Prefix` should
be empty string.

The difference between using AWS SDK `listObjects` and this one:

 * Retries based on the client's retry settings.
 * Supports recursive directory listing.
 * Makes multiple requests if the number of objects to list is greater than 1000.

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `objectsFound`
 * `dirsFound`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when done listing and no more 'data' events will be emitted.
 * `'data' (data)` - emitted when a batch of objects are found. This is
   the same as the `data` object in AWS SDK.
 * `'progress'` - emitted when `progressAmount`, `objectsFound`, and
   `dirsFound` properties change.

And these methods:

 * `abort()` - call this to stop the operation.

### client.deleteObjects(s3Params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#deleteObjects-property

`s3Params` are the same.

The difference between using AWS SDK `deleteObjects` and this one:

 * Retry based on the client's retry settings.
 * Make multiple requests if the number of objects you want to delete is
   greater than 1000.

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when all objects are deleted.
 * `'progress'` - emitted when the `progressAmount` or `progressTotal` properties change.
 * `'data' (data)` - emitted when a request completes. There may be more.

### client.deleteDir(s3Params)

Deletes an entire directory on S3.

`s3Params`:

 * `Bucket`
 * `Prefix`
 * (optional) `MFA`

Returns an `EventEmitter` with these properties:

 * `progressAmount`
 * `progressTotal`

And these events:

 * `'error' (err)`
 * `'end'` - emitted when all objects are deleted.
 * `'progress'` - emitted when the `progressAmount` or `progressTotal` properties change.

`deleteDir` works like this:

 0. Start listing all objects in a bucket recursively. S3 returns 1000 objects
    per response.
 0. For each response that comes back with a list of objects in the bucket,
    immediately send a delete request for all of them.

### client.copyObject(s3Params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#copyObject-property

`s3Params` are the same. Don't forget that `CopySource` must contain the
source bucket name as well as the source key name.

The difference between using AWS SDK `copyObject` and this one:

 * Retry based on the client's retry settings.

Returns an `EventEmitter` with these events:

 * `'error' (err)`
 * `'end' (data)`

### client.moveObject(s3Params)

See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#copyObject-property

`s3Params` are the same. Don't forget that `CopySource` must contain the
source bucket name as well as the source key name.

Under the hood, this uses `copyObject` and then `deleteObjects` only if the
copy succeeded.

Returns an `EventEmitter` with these events:

 * `'error' (err)`
 * `'copySuccess' (data)`
 * `'end' (data)`

## Testing

`S3_KEY=<valid_s3_key> S3_SECRET=<valid_s3_secret> S3_BUCKET=<valid_s3_bucket> npm test`

Tests upload and download large amounts of data to and from S3. The test
timeout is set to 40 seconds because Internet connectivity waries wildly.
