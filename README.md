EvaporateJS
===========

EvaporateJS is a javascript library for directly uploading files from a web browser to AWS S3, using S3's multipart upload. 

### Why?
EvaporateJS can resume an upload after a problem without having to start again at the beginning. For example, let's say
you're uploading a 1000MB file, you've uploaded the first 900MBs, and then there is a problem on the network. Normally
at this point you'd have to restart the upload from the beginning. Not so with EvaporateJS - it will only redo part
that failed, and then carry on from where it left off, and upload the final 100MB.

In addition, EvaporateJS can resume uploads of entire files and parts when the upload failed due to a user or client
error. For example, if the user refreshes the browser during an upload. To reciver, upload the same files. Those
files (or the successfully uploaded file parts) that are available on AWS S3 through an incomplete multipart upload
or an existing S3 object will not be reuploaded.

## Set up EvaporateJS


1. Include evaporate.js in your page

     <script language="javascript" type="text/javascript" src="../evaporate.js"></script>

2. If you want to compute an MD5 digest for AWS, then make sure to include your preferred javascript
cryptography library that supports creating an MD5 digest for the Content-MD5 request header as
specified [here](https://www.ietf.org/rfc/rfc1864.txt). The following library provides support:

     - [Spark MD5](https://github.com/satazor/SparkMD5)

2. Setup your S3 bucket, make sure your CORS settings for your S3 bucket looks similar to what is provided
below (The PUT allowed method and the ETag exposed header are critical).

    The `HEAD` method is required to support resuming failed uploads mid-stream. The `DELETE` method is required to support
    aborting multipart uploads.

        <CORSConfiguration>
            <CORSRule>
                <AllowedOrigin>https://*.yourdomain.com</AllowedOrigin>
                <AllowedOrigin>http://*.yourdomain.com</AllowedOrigin>
                <AllowedMethod>PUT</AllowedMethod>
                <AllowedMethod>POST</AllowedMethod>
                <AllowedMethod>DELETE</AllowedMethod>
                <AllowedMethod>GET</AllowedMethod>
                <AllowedMethod>HEAD</AllowedMethod>
                <ExposeHeader>ETag</ExposeHeader>
                <AllowedHeader>*</AllowedHeader>
            </CORSRule>
        </CORSConfiguration>

3. Setup your S3 bucket Policy to support Aborting and Listings multi-part uploads. To support resumable uploads the AWS account used for uploads
must have the ability to get S3 objects and list multipart uploads and parts. The following AWS S3 policy can be used as a template. Note that the
statement consists of two parts: the first handles the bucket policy; the second handles the objects and multipart uploads within the S3 bucket.

    Replace the AWS ARNs with values that apply to your account and S3 bucket organization.

        {
            "Version": "2012-10-17",
            "Id": "Policy145337ddwd",
            "Statement": [
                {
                    "Sid": "",
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::6681765859115:user/me"
                    },
                    "Action": [
                        "s3:ListBucket",
                        "s3:ListBucketMultipartUploads"
                    ],
                    "Resource": "arn:aws:s3:::mybucket"
                },
                {
                    "Sid": "",
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::6681765859115:user/me"
                    },
                    "Action": [
                        "s3:AbortMultipartUpload",
                        "s3:GetObject",
                        "s3:ListMultipartUploadParts",
                        "s3:PutObject"
                    ],
                    "Resource": "arn:aws:s3:::mybucket/*"
                }
            ]
        }

4. Setup a signing handler on your application server (see `signer_example.py`).  This handler will create a signature for your multipart request that is sent to S3.  This handler will be contacted via AJAX on your site by evaporate.js. You can monitor these requests by running the sample app locally and using the Chrome Web inspector.


## Running the example application

The example application is a simple and quick way to see evaporate.js work.  There are some basic steps needed to make it run locally:

1. Install Google App Engine for Python found [here](https://developers.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python) (The example app is GAE ready and it is run using the GAE dev server)

2. Set your AWS Key and S3 bucket in example/evaporate_example.html. This configuration does not use Md5 Digest verfication.

        var _e_ = new Evaporate({
           signerUrl: '/sign_auth', # Do not change this in the example app
           aws_key: 'your aws_key here',
           bucket: 'your s3 bucket name here',
        });

3. Set your AWS Secret Key in example/signing_example.py

        def get(self):
           to_sign = str(self.request.get('to_sign'))
           signature = base64.b64encode(hmac.new('YOUR_AWS_SECRET_KEY', to_sign, sha).digest())
           self.response.headers['Content-Type'] = "text/HTML"
           self.response.out.write(signature)

4. Run it! (From root of Evaporate directory). and visit 'http://localhost:8080/'

        $ dev_appserver.py app.yaml

5. Upload a file then visit the bucket you specified on the S3 Console page, it will appear there!

## Use EvaporateJS


So far the api contains just two methods, and one property

### new Evaporate()

`var evap = new Evaporate(config)`


`config` has 3 required properties

* **signerUrl**:  a url on your application server which will sign a string with your aws secret key. for example 'http://myserver.com/auth_upload'

* **aws_key**:  your aws key, for example 'AKIAIQC7JOOdsfsdf'

* **bucket**:  the name of your bucket to which you want the files uploaded , for example 'my.bucket.name'


`config` has some optional parameters

* **logging**: default=true, whether EvaporateJS outputs to the console.log  - should be `true` or `false`
* **maxConcurrentParts**: default=5, how many concurrent file PUTs will be attempted
* **partSize**: default = 6 * 1024 * 1024 bytes, the size of the parts into which the file is broken
* **retryBackoffPower**: default=2, how aggressively to back-off on the delay between retries of a part PUT
* **maxRetryBackoffSecs**: default=20, the maximum number of seconds to wait between retries 
* **progressIntervalMS**: default=1000, the frequency (in milliseconds) at which progress events are dispatched
* **aws_url**: default='https://s3.amazonaws.com', the S3 endpoint URL
* **cloudfront**: default=false, whether to format upload urls to upload via CloudFront. Usually requires aws_url to be something other than the default
* **timeUrl',**: default=undefined, a url on your application server which will return a DateTime. for example '/sign_auth/time' and return a RF 2616 Date (http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html) e.g. "Tue, 01 Jan 2013 04:39:43 GMT".  See https://github.com/TTLabs/EvaporateJS/issues/74
* **computeContentMd5',**: default=false, whether to compute and send an MD5 digest for each part for verification by AWS S3.,
* **cryptoMd5Method',**: default=undefined, a method that computes the MD5 digest according to https://www.ietf.org/rfc/rfc1864.txt. Only applicable when `computeContentMd5` is set.
    Method signature is `function (data) { return 'computed MD5 digest of data'; }` where `data` is a JavaScript binary string representation of the body payload to encode. If you are using:
    - Spark MD5, the method would look like this: `function (data) { return btoa(SparkMD5.ArrayBuffer.hash(data, true)); }`. The `data parameter is an `ArrayBuffer`.
* **s3FileCacheHoursAgo',**: default=no cache, whether to use the S3 uploaded cache of parts and files for ease of recovering after
    client failure or page refresh. The value should be a whole number representing the number of hours ago to check for uploaded parts
    and files. The uploaded parts and and file status are retrieved from S3. If no cache is set, EvaporateJS will not resume uploads after
    client or user errors. Refer to the section below for more information on this configuration option.

### .add()

`evap.add(config)`

`config` has 2 required parameters:

* **name**: _String_. the S3 ObjectName that the completed file will have
* **file**: _File_. a reference to the file object

The `.add()` method returns the internal EvaporateJS id of the upload to process. Use this id to abort or cancel an upload.

`config` has 8 optional parameter:


* **xAmzHeadersAtInitiate**, **xAmzHeadersAtUpload**, **xAmzHeadersAtComplete**: _Object_. an object of key/value pairs that represents the x-amz-... headers that should be added to the initiate POST, the upload PUTS, or the complete POST to S3 (respectively) and should be signed by the aws secret key. An example for initiate would be `{'x-amz-acl':'public-read'}` and for all three would be `{'x-amz-security-token':'the-long-session-token'}` which is needed when using temporary security credentials (IAM roles).

* **notSignedHeadersAtInitiate**: _Object_. an object of key/value pairs that represents the headers that should be added to the initiate POST to S3 (not added to the part PUTS, or the complete POST). An example would be `{'Cache-Control':'max-age=3600'}`

* **signParams**: _Object_. an object of key/value pairs that will be passed to _all_ calls to the signerUrl. 

* **complete**: _function(xhr, awsObjectKey)_. a function that will be called when the file upload is complete. Version 1.0.0 introduced the `awsObjectKey` parameter to notifiy the client of the S3 object key that was used if the object already exists on S3.
 
* **cancelled**: _function()_.  a function that will be called when a successful cancel is called for an upload id.

* **info**: _function(msg)_. a function that will be called with a debug/info message, usually logged as well.

* **warn**: _function(msg)_. a function that will be called on a potentially recoverable error, and will be retried (e.g. part upload).

* **error**: _function(msg)_. a function that will be called on an irrecoverable error.

* **progress**: _function(p)_. a function that will be called at a frequency of _progressIntervalMS_ as the file uploads, where _p_ is the fraction (between 0 and 1) of the file that is uploaded. Note that this number will normally increase monotonically, but when a parts errors (and needs to be re-PUT) it will temporarily decrease.

* **contentType**: _String_. the content type (MIME type) the file will have

### .cancel()
`evap.cancel(id)`

`id` is the id of the upload that you want to cancel

### .supported

The `supported` property is _Boolean_, and indicates whether the browser has the capabilities required for Evaporate to work.

### s3FileCacheHoursAgo

When `s3FileCacheHoursAgo` is enabled, the uploader will create a small footprint of the uploaded file in `localStorage.awsUploads`. Before a
file is uploaded, this cache is queried by a key consisting of the file's name, size, mimetype and date timestamp.
It then verifies that the `partSize` used when uploading matches the partSize currenlty in use. To prevent fase positives, the
upload then calcuates the MD5 digest of the first part for final verification.

If the uploaded file has an unfinished multipart upload ID associated with it, then the uploader queries S3 for the parts that 
have been uploaded. It hen uploads only the unfinished parts.

If the uploaded file has no open multipart upload, then the ETag of the last time the file was uploaded to S3 is compared to 
the Etag of what is currently uploaded. If the the two ETags match, the file is not uploaded again.

The timestamp of the last time the part was uploaded is compared against the value of a `Date()` calculated as `s3FileCacheHoursAgo` ago
as a way to gauge 'freshness'. If the last upload was earlier than the number of hours specified, then the file is uploaded again.

It is still possible to have different files with the same name, size and timestamp. In this case, EvaporateJS calculates the checksum for the first
part and compares that to the checksum of the first part of the file to be uploaded. If they differ, the file is uploaded anew.

Note that in order to determine if the uploaded file is the same as a local file, the uploader invokes a HEAD request to S3.
The AWS S3 permissions to allow HEAD also allow GET (get object). This means that your signing url algorithm might want to not sign
GET requests. It goes without saying that your AWS IAM credentials and secrets should be protected and never shared.

### AWS S3 Cleanup and Housekeeping

After you initiate multipart upload and upload one or more parts, you must either complete or abort multipart upload in order to stop 
getting charged for storage of the uploaded parts. Only after you either complete or abort multipart upload, Amazon S3 frees up the parts 
storage and stops charging you for the parts storage. Refer to the 
[AWS Multipart Upload Overview]http://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html for more information.

The sample S3 bucket policy shown above should configure your S3 bucket to allow cleanup of orphaned multipart uploads but the cleanup task is
not part of EvaporateJS. A separate tool or task will need to be created to query orphaned multipart uploads and abort them using some appropriate 
heuristic.

Refer to this functioning [Ruby on Rails rake task]https://github.com/bikeath1337/evaporate/blob/master/lib/tasks/cleanup.rake for ideas.  


## Integration

* [angular-evaporate](https://github.com/uqee/angular-evaporate) &mdash; AngularJS module.
* 

## License

EvaporateJS is licensed under the BSD 3-Caluse License
http://opensource.org/licenses/BSD-3-Clause

## Working with temporary credentials in Amazon EC2 instances

* [Security and S3 Multipart Upload](http://www.thoughtworks.com/mingle/infrastructure/2015/06/15/security-and-s3-multipart-upload.html)
