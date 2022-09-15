# Week 5 : Orchestration Services

## Agenda
- Introduction to EventBridge, SQS, SNS, other lambda triggers - 30 minutes
- Introduction to Step functions - 30 minutes
- Guest speaker(s) - 1 hour
- Class assignments - 1 hour

### 1. Interfacing with storage

There is one storage events we would like to focus on: `New object created event`. It will be used in the homework assignment.  To listen to this type of event, we will need to create a lambda once again with SAM.

Create a new lambda through the SAM cli init command. This time you want to use the app-template called `5 - Quick Start: S3`. This will generate a lambda, which is invoked when new objects is created in a specified bucket. Additionally, a folder is created with custom event files we can use for local development.

Remember to install the libraries with

`npm install`

Then we can run the lambda with

`sam local invoke --event events/event-s3.json`.

However, you will face an `access denied` error. This is because the event is pointing to an example **bucket** and **object** we do not have access to. By inspecting the code:

```javascript
exports.s3JsonLoggerHandler = async (event, context) => { // we now have an event payload!
  const getObjectRequests = event.Records.map((record) => {
    const params = {
      Bucket: record.s3.bucket.name,
      Key: record.s3.object.key,
    };

    ... // do something with the event
};
```

we see that is taking the records, specifically object and name values of our `event.json` file

```json
{
  "Records": [
    {
      ...,
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "example-bucket",  <---- change to own bucket name
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws:s3:::example-bucket"
        },
        "object": {
          "key": "test/key", <---- change to own object/filename
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "sequencer": "0A1B2C3D4E5F678901"
        }
      }
    }
  ]
}
```

For a complete list of S3 events see (here)[https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html])

### 2. Deploying the lambda

We will deploy this almost the same way as before, except we will need to specifiy a bucket in our deployment step - in this case the bucket to listen to will be the bucket you configured in week 3 for access logging.

First, we can build with `sam build`. But by looking at the generated `buildspec.yml` we see that sam build is just a shortcut to the command for:

`aws cloudformation package --template template.yml --s3-bucket $S3_BUCKET --output-template template-export.yml`

This time however, we need to provide a parameter for a bucket we want our lambda to listen to. This is specificed in our cloudformation template:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  S3 Events lambda

Parameters:
  AppBucketName:
    Type: String
    Description: "REQUIRED: Unique S3 bucket name to use for the app."

Resources:
  S3JsonLoggerFunction:
    Type: AWS::Serverless::Function
    Properties:
      ...
      Events:
        S3NewObjectEvent:
          Type: S3
          Properties:
            Bucket: !Ref AppBucket
            Events: s3:ObjectCreated:*

  AppBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref AppBucketName
```

Deploying this will create our usual lambda with an event configuration for new objects created. But it will also create a new bucket with the name we provide to it. This illustrates how to create multiple resources as part of our file. Clever! We will go more into depth with this in just a second.

### 3. Step Functions

TODO