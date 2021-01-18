---
type: exercise
slug: make-aws-less-painful-with-lambda
guid: https://pasztor.at/blog/make-aws-less-painful-with-lambda/
authors:
- janos
categories:
- Technology
date: "2018-06-12T00:00:00Z"
publishDate: "2018-06-12T00:00:00Z"
summary: Amazon Web Services can sometimes be a royal pain in the backside, especially because a lot of their services are not entirely feature complete. A lot of these gaps can, however, be filled with Amazon Lambda.
images:
- posts/make-aws-less-painful-with-lambda.png
preview: posts/make-aws-less-painful-with-lambda.jpg
tags:
- DevOps
- Cloud
title: Make AWS less painful with Lambda functions
---

All right, let me explain what I mean. Amazon has a wide range of services, but most of their services are designed
to be used in one specific way and if you don't use them the way it was designed, you very soon discover the limitations
of the service.

To name a few examples ECS didn't have a service discovery feature for a long time, or if you are using the cheaper spot
instances, handling the spot instance shutdown notification requires you to write your own code.

Now, deploying your own, long running code presents its own challenges. It's not impossible, just challenging.

## What is Lambda?

This is where Lambda comes into play. At its core, Amazon Lambda lets you run tiny pieces of code in reaction to events
from CloudWatch (health warnings), SNS (notification system), SQS (queue system) and with Lambda@Edge even on the CDN
nodes.

So when an event happens, Lambda will start a, say, NodeJS container and runs the function you have specified. You can
then write data into your database, or even use the AWS API to change things. You don't have to stick with NodeJS
either, at the time of writing you can go with Python, Java 8, C# (.NET Core) and Go.

From a billing side, you are paying for 100ms increments of run time, and for used memory as well.

Now, before you jump into the Amazon web interface and start writing Lambda functions like crazy, it does come with
a couple of caveats.

First of all, in order to use Lambda you have to set up various IAM roles, as well as manage your code, so you WILL need
some sort of a configuration management tool. In this example I'm going to demonstrate the deployment process using
Ansible, but you should be able to easily port the process to Terraform or other management tools.

Second, if you want real time applications, such as reacting to HTTP requests real time, be aware that there is no 
execution time guarantee. Most events will complete pretty fast, but if you haven't had an event for a while, some 
events, in my experience, take up to one second to run even the simplest code.

## Writing your lambda code

As a basic setup I recommend you create a new project in your IDE of choice and put everything you need for your Lambda
function in a separate folder. Your root function will be a file that you can name as you wish, say `handler.js`. In
addition to that you can also place other files, such as `node_modules` inside that folder. When we upload this folder,
we'll ZIP it together and publish it that way.

Now, your code will look something like this:

```javascript
exports.handler = function(event, context, callback) {
    
}
``` 

The first parameter, obviously, contains details of the event. We'll take a look at that in detail a little later. The
second parameter is the [NodeJS context object](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-prog-model-context.html).

The third parameter, however, is a tricky one. Depending on what invokes your function, the call may be either synchronous
or asynchronous. Most of the time you won't care because you don't need to send any data back, but if you do (for example
with Lambda@Edge) the callback function comes in handy. For synchronous calls you can just `return` the data.

An other cool thing you can do in your code is use the `console.log()` function, which will end up in your CloudWatch
logs, helping you debug potential issues.

You can also use the AWS SDK without installing it as a dependency:

```javascript
var AWS = require('aws-sdk');
var ec2 = new AWS.EC2({apiVersion: '2016-11-15'});
ec2.describeInstances(
    //...
);
```

## Deploying your code

Now, as I mentioned, it is not recommended to simply deploy your code from the web interface, and you'll see why in a
moment. As I mentioned, I'll use Ansible to deploy the code, which takes configuration in YAML format.

First of all, let's deploy our Lambda function. As our first task we'll ask Ansible to pack up our folder into a ZIP
file:

```yaml
- hosts: localhost
  tasks:
    - name: "Packing up lambda code..."
      archive:
        dest: /tmp/lambda.zip
        path: files/lambda/
        format: zip
```

This will pack up the contents of `files/lambda` into `/tmp/lambda.zip`. So far so good. Next, we need to set up 
an IAM role so our Lambda function has permissions, for example to send the logs to CloudWatch. For that we'll generate
an IAM role:

```yaml
{% raw %}- hosts: localhost
  tasks:
    #...
    - name: "Creating lambda IAM role..."
      iam_role:
        name: my-lambda
        assume_role_policy_document: "{{lookup('template', 'files/lambda-assume-role-policy.json.j2')}}"
        managed_policy: []{% endraw %}
```

Now, as you can see, we don't have any managed policies, but we need to supply the role with an assume role policy
document. This document tells AWS when to allow a service to use this role. This document is very simple:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Ok, so now we need to add some permissions to that role so we can at least write some logs:

```yaml
{% raw %}- hosts: localhost
  tasks:
    #...
    - name: "Creating lambda IAM policy..."
      iam_policy:
        iam_type: role
        iam_name: "my-lambda"
        policy_name: "my-lambda"
        state: present
        policy_json: " {{ lookup( 'template', 'files/lambda-policy.json.j2') }} "{% endraw %}
```

The policy starts out quite simple, but if you need your Lambda function to access more AWS APIs, you can add permissions
here:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
```

**Be careful!** If you grant your Lambda function excessive permissions, you'll end up with a potential security hole!
Make sure you lock down your system!

Now we can actually deploy the Lambda function:

```yaml
- hosts: localhost
  tasks:
    #...
    - name: "Creating lambda function..."
      lambda:
        name: my-lambda
        description: "Does this and that"
        region: us-east-1
        role: "my-lambda"
        handler: handler.handler
        runtime: nodejs8.10
        zip_file: /tmp/lambda.zip
        timeout: 15
```

A few things to note here. First of all, your handler is going to be comprised of your file name, and the function
name separated with a dot. So `handler.js` and `exports.handler` will result in `handler.handler` in the specification.

Second, if your Lambda function needs access to your VPC to access your database or APIs, you will need to specify the
`vpc_subnet_ids` and `vpc_security_group_ids` parameters. You can find more infos on that in the
[Ansible documentation](https://docs.ansible.com/ansible/latest/modules/lambda_module.html).

## Triggering your function

After you have deployed your Lambda function, you need to set up a trigger. In our example we'll take a CloudWatch Events
event to trigger the function.

CloudWatch, by default, can't access your function. To enable that we need to create an IAM role that allows CloudWatch.
Same as above, we create the IAM role:

```yaml
{% raw %}- hosts: localhost
  tasks:
    #...
    - name: "Creating CloudWatch IAM role..."
      iam_role:
        name: my-cloudwatch
        assume_role_policy_document: "{{lookup('template', 'files/cloudwatch-assume-role-policy.json.j2')}}"
        managed_policy: []{% endraw %}
```

The policy is similar:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

And here go the permissions:

```yaml
{% raw %}- hosts: localhost
  tasks:
    #...
    - name: "Creating CloudWatch IAM policy..."
      iam_policy:
        iam_type: role
        iam_name: "my-cloudwatch"
        policy_name: "my-cloudwatch"
        state: present
        policy_json: " {{ lookup( 'template', 'files/cloudwatch-policy.json.j2') }} "{% endraw %}
```

Finally, here's the permissions for CloudWatch:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:your-region:your-account-id:function:my-lambda",
            "Condition": {
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:events:your-region:your-account-id:rule/my-cloudwatch-rule"
                }
            }
        }
    ]
}
```

Notice that we locked down this rule to ONLY be applied if the specific CloudWatch rule was used to call the function.
This, again, goes along the lines of locking your IAM permissions down.

Now, as a final piece we need to create our CloudWatch rule to trigger our Lambda function:

```yaml
- hosts: localhost
  tasks:
    #...
    - name: "Creating CloudWatch event rule..."
      cloudwatchevent_rule:
        name: my-cloudwatch-rule
        description: "Does this and that"
        role_arn: "arn:aws:iam::your-account-id:role/my-cloudwatch"
        event_pattern: |
          TBD: ADD EVENT PATTERN HERE
        targets:
          - id: my-lambda
            arn: "arn:aws:lambda:your-region:your-account-id:function:my-lambda"
```

Where do you get your event pattern from you ask? Simple. When you go into the [CloudWatch Events rule creator](https://console.aws.amazon.com/cloudwatch/home?#rules:action=create)
the interface will let you click together your desired rule set and displays the JSON that you need to add to your
Ansible configuration.

## What's next?

Ok, so now you have your rules all set up, and you can also test your events firing by looking a the monitoring
interface in CloudWatch Events. You can, of course, add `console.log` to your code to log the data you received into
[CloudWatch Logs](https://console.aws.amazon.com/cloudwatch/home#logs:), which should make writing code easier.

Alternatively, you could also [read the documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html)
to find out how the event data structures look. (I don't say, right?) At the end of the day, you may need to grant
some additional permissions to your function to make it really useful.

To give you some ideas what to try out, you could set up an ECS cluster and run it on spot instances. You could then 
write a Lambda function that reacts to the spot instance termination and evicts the ECS instance beforehand so there
is no service interruption.

Enjoy!