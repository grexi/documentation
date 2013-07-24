# Python Amazon S3 integration

[Amazon S3](http://aws.amazon.com/s3/) is a Storage-as-a-Service solution. It provides a simple web service interface that can be used to store and retrieve any amount of data, at any time, from anywhere on the web.

## Amazon S3 SDK

There are a few python SDKs for Amazon S3, the first being the official one:
* [Amazon S3 Python SDK / boto](http://aws.amazon.com/sdkforpython/)
* [python-s3](https://github.com/nephics/python-s3)
* [py-mini-s3](http://code.google.com/p/pts-mini-gpl/source/browse/#svn/trunk/py-mini-s3)

## Getting started

Follow the [Amazon Guide](http://docs.aws.amazon.com/AmazonS3/latest/gsg/GetStartedWithS3.html) to setup an account and get the [AWS access credentials](http://aws.amazon.com/security-credentials).

To use `AWS S3` storage in your application just specify additional dependency in your `requirements.txt`:

~~~
boto==2.9.8
~~~

## Example usage:

First create an AWS account and make your credentials accessible by your application. The recommended way is to provide them via environment variables. To do this, use [Config Add-on](https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Deployment/Custom%20Config):

~~~bash
$ cctrlapp APP_NAME/default addon.add config.free --AWS_SECRET_KEY=[YOUR_SECRET_KEY] --AWS_ACCESS_KEY=[YOUR_ACCESS_KEY]
~~~

Now try out some operations on buckets and objects:

~~~python

import uuid
import boto
from boto.s3.key import Key

if __name__ == "__main__":

    # S3 client connection - AWS credentials by default are read
    # from env varaiables: AWS_ACCESS_KEY and AWS_SECRET_KEY
    conn = boto.connect_s3()

    # Create bucket
    bucket = conn.create_bucket('testbucket{0}'.format(str(uuid.uuid4())))

    # List buckets
    buckets = conn.get_all_buckets()
    print 'Buckets: ',buckets

    # Put object
    k = Key(bucket)
    k.key = 'key'
    file_name = 'testfile{0}'.format(str(uuid.uuid4()))
    file = open(file_name, 'w')
    file.write('This is a test file which will land on S3')
    file.close()
    k.set_contents_from_filename(file_name)

    # Read object
    key = bucket.get_key('key')

    print key.get_contents_as_string()

    # Delete object
    bucket.delete_key('key')

    # Delete bucket
    conn.delete_bucket(bucket.name)
~~~