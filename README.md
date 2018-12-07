# Demo AWS CLI Cloudwatch Lambda

This demo is based on this [AWS blog post](https://aws.amazon.com/blogs/compute/building-a-dynamic-dns-for-route-53-using-cloudwatch-events-and-lambda/)


## Preflight

Verify you are signed in:

```sh
aws opsworks describe-my-user-profile
```


## Create a role
 
Create a role, then look at the output to see the ARN:

```sh
aws iam create-role \
--role-name demo-role \
--assume-role-policy-document file://demo-trust.json
```

Response:

```json
{
  "Role": {
    "Path": "/",
    "RoleName": "demo-role",
    "RoleId": "LVJPXFVUTDXRITBHQZVKGZ",
    "Arn": "arn:aws:iam::048251220134:role/demo-role",
    "CreateDate": "2018-12-07T15:41:19Z",
    "AssumeRolePolicyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "",
          "Effect": "Allow",
          "Principal": {
            "Service": "lambda.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
  }
}
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:role/demo-role
```

This is the "role ARN", and you will need it later.


## Create a policy

Create a policy, then look at the output to see the ARN:

```sh
aws iam create-policy \
--policy-name demo-policy \
--policy-document file://demo-policy.json
```

Response:

```json
{
  "Policy": {
    "PolicyName": "demo-policy",
    "PolicyId": "RLVJRZWDQZGSIIHWDEYDWB",
    "Arn": "arn:aws:iam::048251220134:policy/demo-policy",
    "Path": "/",
    "DefaultVersionId": "v1",
    "AttachmentCount": 0,
    "IsAttachable": true,
    "CreateDate": "2018-12-07T15:41:57Z",
    "UpdateDate": "2018-12-07T15:41:57Z"
  }
}
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:policy/demo-policy
```
This is the "policy ARN", and you will need it later.


## Attach

Attach the role to the policy:

```sh
aws iam attach-role-policy \
--role-name demo-role \
--policy-arn arn:aws:iam::048251220134:policy/demo-policy
```

Response is empty.


## Create a script

Write a function, for example a python script that says hello.

```py
def lambda_handler(event, context):
    print("Hello World")
```

Create a file `HelloWorld.py` with the function above.


Create a zip file that contains the function, which is necessary to upload to AWS:

```sh
zip HelloWorld.zip HelloWorld.py
```


## Create a function

Create a function by using the AWS CLI:

```sh
aws lambda create-function \
--function-name HelloWorld \
--runtime python3.6 \
--role arn:aws:iam::048251220134:role/demo-role \
--handler HelloWorld.lambda_handler \
--zip-file fileb://HelloWorld.zip
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:function:HelloWorld
```
This is the "function ARN", and you will need it later.


## Invoke a function

Invoke:

```sh
aws lambda invoke \
--function-name HelloWorld output.txt
```

Response:

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

Verify the output text:

```sh
cat output.txt
```

Should be:

```txt
null
```


## Create a CloudWatch events rule

Create a CloudWatch events rule, then look at the output to see the ARN:

```sh
aws events put-rule \
--name demo-rule \
--schedule-expression 'rate(1 minute)'
```

Response:

```json
{
    "RuleArn": "arn:aws:events:us-east-1:048251220134:rule/demo-rule"
}
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:rule/demo-rule
```

This is the "rule ARN", and you will need it later.


### Create permission

Create a permission:

```sh
aws lambda add-permission \
--function-name HelloWorld \
--statement-id demo-statement \
--action 'lambda:InvokeFunction' \
--principal events.amazonaws.com \
--source-arn arn:aws:iam::048251220134:rule/demo-rule
```

Reponse:

```json
{
  "Statement": "{\"Sid\":\"demo-statement\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"events.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-east-1:904129017616:function:HelloWorld\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:iam::048251220134:rule/demo-rule\"}}}"
}
```

## Create a target

Create a target:

```sh
aws events put-targets \
--rule demo-rule \
--targets file://demo-targets.json
```

Response:

```json
{
    "FailedEntryCount": 0,
    "FailedEntries": []
}
```


## Verify in the console

If you like, you can use the AWS console to see the items.


### See roles and policies

Browse https://console.aws.amazon.com/iam/home?region=us-east-1#/roles

You see all your roles, including demo-role.

Browse https://console.aws.amazon.com/iam/home?region=us-east-1#/roles/demo-role

You see the demo-role, with tabs and information including:

* Permissions

  * Policy name: demo-policy 

* Trust relationships

  * Trusted entities: The identity provider(s) lambda.amazonaws.com

The demo-role shows the demo-policy, which means the demo-role and demo-policy are attached successfully.


### See functions

Browse https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions

You see all your functions, including HelloWorld: 

* Function name: HelloWorld
* Runtime: Python 3.6
* Code size: 237 bytes

Browse https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions/HelloWorld?tab=graph

You see the HelloWorld function, including the "Function code" area that shows the source code:

```python
def lambda_handler(event, context):
    print("Hello World")
```


### See rules

Browse https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#rules

You see all your rules, including demo-rule.

Browse https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#rules:name=demo-rule

You see the rule, something like this:

* Schedule: Fixed rate of 1 minutes
* Status: Enabled
* Targets
  * Type: Lambda function
  * Name: HelloWorld  
  * Constant: {"Domain": "mydomain.com","MasterDns": "10.0.0.1","ZoneId": "AA11BB22CC33DD","IgnoreTTL": "False","ZoneSerial": ""} 


## See logs

Browse https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logs:

You see all your logs, including /aws/lambda/HelloWorld.

Browse https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/HelloWorld

You see the HelloWorld "Log Streams". 

Click on the most-recent stream.

You see the log text, something like this:

```text

15:45:02 START RequestId: 0f5369ea-fa37-11e8-9245-17324be3090c Version: $LATEST
15:45:02 Hello World
15:45:02 END RequestId: 0f5369ea-fa37-11e8-9245-17324be3090c
15:45:02 REPORT RequestId: 0f5369ea-fa37-11e8-9245-17324be3090c Duration: 0.60 ms Billed Duration: 100 ms Memory Size: 128 MB Max Memory Used: 21 MB
```


## Credit & Thanks

Links:

* ["Building a dynamic DNS for Route 53" - AWS blog post](https://aws.amazon.com/blogs/compute/building-a-dynamic-dns-for-route-53-using-cloudwatch-events-and-lambda/)

* ["AWS Lambda: Programatically create a Python 'Hello World' function" - blog post - by Mark Needham](https://markhneedham.com/blog/2017/04/02/aws-lambda-programatically-create-a-python-hello-world-function/)

* ["Route 53 hosted zone management" - Atomic Object blog post - By Justin Kulesza](https://spin.atomicobject.com/2016/04/28/route-53-hosted-zone-managment/)

* ["Powering Secondary DNS in a VPC using AWS Lambda and Amazon Route 53 Private Hosted Zones" - AWS blog post - By  Bryan Liston](https://aws.amazon.com/blogs/compute/powering-secondary-dns-in-a-vpc-using-aws-lambda-and-amazon-route-53-private-hosted-zones/)
