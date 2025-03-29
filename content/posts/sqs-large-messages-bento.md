---
author: "Jem Davies"
title: "SQS Extended Client & Bento"
date: "2025-03-26"
description: "What the heck is the SQS Extended Client? - and how you can use Bento to process such eclectic things..."
tags: ["AWS", "SQS", "Bento", "Python", "Terraform"]
ShowToc: true
ShowBreadCrumbs: false
---

---

## Intro

Standard SQS messages are limited in size to 256 KB :scream:  - but you can use the [Amazon SQS Extended Client](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-managing-large-messages.html) to send larger messages up to 2 GB. These libraries store the messages larger than 256 KB in S3 and send a reference to the stored object in the SQS queue.

Amazon provides two client libraries, for [Python](https://github.com/awslabs/amazon-sqs-python-extended-client-lib/) and [Java](https://github.com/awslabs/amazon-sqs-java-extended-client-lib). There are a few unofficial ones knocking around on github: [Javascript](https://github.com/dvla/sqs-extended-client) and [Go](https://github.com/co-go/sqs-extended-client-go).

---

## Setup using Terraform

Ok so, we only need a couple of AWS resources:

 - An SQS Queue 
 - An S3 Bucket

This terraform configuration will create these resources: 

```hcl
provider "aws" {
  region = "eu-west-2"
}

resource "random_string" "bucket_suffix" {
  length  = 6
  special = false
  upper   = false
}

resource "aws_s3_bucket" "large_message_bucket" {
  bucket = "large-message-bucket-${random_string.bucket_suffix.result}"
  force_destroy = true
}

resource "aws_sqs_queue" "large_message_queue" {
  name = "large-message-queue"
}

output "bucket_name" {
  value = aws_s3_bucket.large_message_bucket.id
}

output "queue_url" {
  value = aws_sqs_queue.large_message_queue.url
}
```

After running `terraform apply` you should see in your terminal something like:

```
bucket_name = "large-message-bucket-xxxxxx"
queue_url = "https://sqs.<AWS_REGION>.amazonaws.com/<AWS_ACCOUNT_ID>/large-message-queue"
```

---

## Send a large message using Python

 - Get the dependencies, `boto3` & `amazon-sqs-extended-client`
 - Update the variables `bucket_name` & `queue_url`

```python
import boto3 # pip install boto3
import sqs_extended_client # pip install amazon-sqs-extended-client

# From Terraform Outputs
bucket_name = "large-message-bucket-xxxxxx"
queue_url = "https://sqs.<AWS_REGION>.amazonaws.com/<AWS_ACCOUNT_ID>/large-message-queue"

# Set the Amazon SQS extended client configuration with large payload.
sqs_extended_client = boto3.client("sqs", region_name="<AWS_REGION>")
sqs_extended_client.large_payload_support = bucket_name 
sqs_extended_client.use_legacy_attribute = False

# Sending a large message
small_message = "s"
large_message = small_message * 300000 # Shall cross the limit of 256 KB

send_message_response = sqs_extended_client.send_message(
    QueueUrl=queue_url,
    MessageBody=large_message
)

response_code = send_message_response["ResponseMetadata"]["HTTPStatusCode"]
print(f"send_message_response code: {response_code}")
```

The above Python is largely a modified version of [this](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/extended-client-library-python.html#extended-client-library-python-code-example), but with the managing of the AWS Resources 
removed.

---

## Receive SQS Messages using Bento

[Bento](https://bento.dev) is a data-streaming tool that makes "Fancy stream processing operationally mundane". 

You can use it to implement different types of streams like multiplexing, windowing and enrichments.
It has a number of sources, sinks and processors all of which are yaml config driven - making it versatile and easy to use.

Below is a config that will read from an SQS queue - and for those messages that contain pointers to S3: it will dereference and replace the message contents with the S3 object!

```yaml
input:
  label: sqs_extended_client

  aws_sqs:
    url: "https://sqs.<AWS_REGION>.amazonaws.com/<AWS_ACCOUNT_ID>/large-message-queue"

  processors:
    - switch: 
        # check it's a large message:
        - check: this.0 == "software.amazon.payloadoffloading.PayloadS3Pointer" 
        # use the aws_s3 processor to download it 
          processors:
            - aws_s3:
                bucket: ${! this.1.s3BucketName }
                key: ${! this.1.s3Key }

output:
  stdout: {}
```

Here we are using the [aws_sqs](https://warpstreamlabs.github.io/bento/docs/components/inputs/aws_sqs) input - we have one field defined `url`, with the URL of the queue we created earlier.

Bento has a few different ways you can connect to AWS - they are outlined on the [cloud credentials guide for aws](https://warpstreamlabs.github.io/bento/docs/guides/cloud/aws).

When testing this out, I just set the environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` - [The underlying Go library will look for credentials in all the usual places.](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html)

### Dereferencing the SQS Large Messages

Messages that exceeded the 256 KB limit will look like this: 

```json
[
    "software.amazon.payloadoffloading.PayloadS3Pointer",
    {
        "s3BucketName": "large-message-bucket-xxxxxx",
        "s3Key": "dda3f51a-323a-4424-b2b9-804e8675c6d4"
    }
]
```

So first we use a [switch](https://warpstreamlabs.github.io/bento/docs/components/processors/switch) processor to perform a conditional branch, checking for the existence of `"software.amazon.payloadoffloading.PayloadS3Pointer"` - then we use the [aws_s3](https://warpstreamlabs.github.io/bento/docs/components/processors/switch) to download the contents of the s3 object.

We use [bloblang interpolation](https://warpstreamlabs.github.io/bento/docs/configuration/interpolation/#bloblang-queries) to get the value of `s3_bucket` and `s3_key`. For example `${! this.1.s3Key }` this bloblang query is pretty simple - it is just a key path with array indexes and field names of json objects. Bloblang is a mapping language that you can find more about on the [walkthrough](https://warpstreamlabs.github.io/bento/docs/guides/bloblang/walkthrough/).

Notice that we have defined a set of `processors` within the `input` layer of the stream, this means that only the messages that are created by this `input` are processed by these `processors`. We could have included these steps inside a `pipeline.processors` part of the config but if we were to add more inputs later, we might not want to perform "s3 dereferencing" on the messages that are produced by future inputs. This way we can couple the logic of "s3 dereferencing" with the `aws_sqs` input labelled `sqs_extended_client`. :nerd_face:

The final part of the config will output the messages to `stdout` - this might not be that useful so take a peak at some other options [here](https://warpstreamlabs.github.io/bento/docs/components/outputs/about#categories)!

Happy Streaming :rocket:

---