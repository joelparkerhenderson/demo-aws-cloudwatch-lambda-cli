# Demo AWS Cloudwatch Lambda

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
--role-name demo-aws-cloudwatch-lambda-role \
--assume-role-policy-document file://demo-trust.json
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:role/demo-aws-cloudwatch-lambda-role
```

This is the "role ARN", and you will need it later.


## Create a policy

Create a policy, then look at the output to see the ARN:

```sh
aws iam create-policy \
--policy-name demo-aws-cloudwatch-lambda-policy \
--policy-document file://demo-policy.json
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:policy/demo-aws-cloudwatch-lambda-policy
```
This is the "policy ARN", and you will need it later.


## Attach

Attach the role to the policy:

```sh
aws iam attach-role-policy \
--role-name demo-aws-cloudwatch-lambda-role \
--policy-arn "arn:aws:iam::048251220134:policy/demo-aws-cloudwatch-lambda-policy"
```


## Create a function

Write a function, for example a python script that says hello.

```py
def lambda_handler(event, context):
    print("Hello World")
```

Create a file `HelloWorld.py` with the function above.


### Create a zip

Create a zip file that contains the function, which is necessary to upload to AWS:

```sh
zip HelloWorld.zip HelloWorld.py
```


### Create a function

Create a function by using the AWS CLI:

```sh
aws lambda create-function \
--function-name HelloWorld \
--runtime python3.6 \
--role arn:aws:iam::048251220134:role/demo-aws-cloudwatch-lambda-role \
--handler HelloWorld.lambda_handler \
--zip-file fileb://HelloWorld.zip
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:policy/demo-aws-cloudwatch-lambda-function
```
This is the "function ARN", and you will need it later.


### Invoke

Invoke:

```sh
aws lambda invoke \
--function-name HelloWorld output.txt
```

Verify the output text:

```sh
cat output.txt
```

The output should be:

```txt
null
```

Troubleshooting:

* See our related demo: https://github.com/joelparkerhenderson/demo_aws_lambda_function_hello_world_as_python


## Create a CloudWatch events rule

Create a CloudWatch events rule, then look at the output to see the ARN:

```sh
aws events put-rule \
--name demo-aws-cloudwatch-lambda-rule \
--schedule-expression 'rate(15 minutes)'
```

The output ARN will look something like this:

```
arn:aws:iam::048251220134:policy/demo-aws-cloudwatch-lambda-rule
```

This is the "rule ARN", and you will need it later.


### Create permission

Create a permission:

```sh
aws lambda add-permission \
--function-name demo-aws-cloudwatch-lambda- \
--statement-id Scheduled01 \
--action 'lambda:InvokeFunction' \
--principal events.amazonaws.com \
--source-arn arn:aws:iam::048251220134:policy/demo-aws-cloudwatch-lambda-rule
```

### Create a target

Create a target:

```sh
aws events put-targets \
--rule demo-aws-cloudwatch-lambda-rule \
--targets file://demo-targets.json
```


## Credit & Thanks

Links:

* ["Building a dynamic DNS for Route 53" - AWS blog post](https://aws.amazon.com/blogs/compute/building-a-dynamic-dns-for-route-53-using-cloudwatch-events-and-lambda/)

* ["AWS Lambda: Programatically create a Python 'Hello World' function" - blog post - by Mark Needham](https://markhneedham.com/blog/2017/04/02/aws-lambda-programatically-create-a-python-hello-world-function/)

* ["Route 53 hosted zone management" - Atomic Object blog post - By Justin Kulesza](https://spin.atomicobject.com/2016/04/28/route-53-hosted-zone-managment/)
