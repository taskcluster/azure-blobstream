taskcluster-azure-blobstream
==================

Stream interface built on top of azure for incrementally pushing buffers
and committing them. Designed for "live" logging and bursty streams of
data (which is expected to end eventually).

There are many common cases which the azure client handles _much_ better
then this library if you doing any of the following use the azure
client:

  - writing never ending data (that may be rolled over)
  - randomly accessing or updating blocks/pages
  - uploading files already on disk
  - uploading a stream indefinitely

## Strategy

The algorithm is very simple (dumb)

 - let node stream handle buffering/backpressure
 - write block (BlobBlock) and commit it in same write operation (_write in node streams)
 - now that block is readable
 
Due to how node streams work while we are writing the readable side will buffer its writes up to the high water mark.


## Note about performance for node 0.10

This stream does not do any special internal buffering to optimize
backed up writes. What this means is if your goal is to "append" as fast
as possible without much concern for memory node 0.10 will be _much_
slower then 0.11 because writes are done in order and each buffer is
written sepeartely without merges of the buffers that have yet to be
written. Node 0.11 introduces `_writev` which is used here to merge any
pending buffers before the write which avoids most latency issues.

## Example

```js
var AzureStream = require('taskcluster-azure-blobstream');

var azure = require('azure');
var blobService = azure.createBlobService();

var azureWriter = new AzureStream(
  blobService,
  'mycontainer',
  'myfile.txt'
);

// any kind of node readable stream here
var nodeStream;

nodeStream.pipe(azureWriter);
azureWriter.once('finish', function() {
  // yey data was written
  // get the url
  console.log(blobService.getBlobUrl('mycontainer', 'myfile.txt'));
});

```

## RANDOM NOTES

the `azure-storage` module is slow to load (147ms) and takes up 19mb of
memory (as of 0.3). We don't use very many azure blob api calls so
ideally we could extract (or help the primary lib extract) the url
signing part of authentication into its own lib and then just directly
call http for our operations... The ultimate goal here is to consume
around 5mb (including https overhead) of memory and load in under 20ms.

To correctly consume the url from azure the `x-ms-version` header must
be set to something like `2013-08-15` this allows open ended range
requests (`range: byte=500-`). In combination with etags (and if
conditions) we can build a very fast client (even a fast polling
client).
