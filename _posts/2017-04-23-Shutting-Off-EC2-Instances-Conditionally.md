---
layout: post
title: "Conditionally stopping AWS EC2 Instances Using AWS Lambda, Python, and Serverless"
date: 2017-Apr-23
---

_In an effort to better document learning I do, I'd like to write semi-frequent blog posts detailing solutions to problems I encounter each day. This is the first of those posts._

# The Need

At work, we deal with very large sets of data relating to health care. Developers testing new features frequently create environments in AWS (called "fruits" internally) that can house the (sometimes large) databases and web front-end(s) to run the application on an environment that's similar to our production set-up. At any given time, there are between 30-50 EC2 instances running, all relating to different features or for testing purposes. We wanted a solution that would allow developers to conditionally and automatically stop our instances during the evening to save on costs.

Using [Lambda](https://aws.amazon.com/lambda/?sc_channel=PS&sc_campaign=acquisition_US&sc_publisher=google&sc_medium=lambda_b&sc_content=lambda_e&sc_detail=aws%20lambda&sc_category=lambda&sc_segment=73823576562&sc_matchtype=e&sc_country=US&s_kwcid=AL!4422!3!73823576562!e!!g!!aws%20lambda&ef_id=WPzNgQAABMh75wRS:20170423155129:s), a Python runtime, and [Severless](https://serverless.com), our team created a solution that iterates through instances in our AWS inventory, checks for the presence of a key (called `shutOff`), and stops those instances if the value of the key is set to `true`.

# The Solution

The Python script below is the solution that runs on AWS Lambda, and uses the [boto](http://boto3.readthedocs.io/en/latest/#) Python library:

```
import boto3


def shut_off(event, context):
    """
    shuts off ec2 instances that have a value of 'true' for the tag 'shutOff'
    """
    ec2 = boto3.client('ec2')
    asclient = boto3.client('autoscaling')

    # get reservations that contain instances that have a tag of 'shutOff'
    reservations = ec2.describe_instances(
        Filters=[
            {
                'Name': 'tag:shutOff',
                'Values': [
                    'true'
                ]
            }
        ]
    ).get('Reservations', [])

    instances = sum([[i for i in r['Instances']] for r in reservations], [])

    for i in instances:
        # suspend autoscaling groups
        for j in i['Tags']:
            if j['Key'] == 'aws:autoscaling:groupName':
                asclient.suspend_processes(
                    AutoScalingGroupName=j['Value'],
                    ScalingProcesses=[
                        'Launch',
                        'Terminate',
                        'HealthCheck',
                        'ReplaceUnhealthy',
                        'AZRebalance'
                    ])
        ec2.stop_instances(
            InstanceIds=[i['InstanceId']]
        )
```

One interesting fold in our environments is that we are set up to rapidly take on load, thanks to EC2's ability to autoscale conditionally. As a result, I found that if I simply shut off the instances with the `shutOff` tag, the instance would be considered "unhealthy", and would subsequently be terminated and recreated. In addition to stopping the instances, I needed to first suspend the autoscaling processes on the instances that check for an "unhealthy" instance.

To deploy the script to Lambda, I used Serverless. This was my first foray into using Serverless, and it's really slick. Essentially, you supply a YAML file that acts as the configuration for the Lambda event. Then, in a terminal, you can simply type `sls deploy` to create a fully-configured Lambda event.

# What I Learned
This was my first attempt at writing Python (a frequently-cited "competitor" of my beloved Ruby), and I learned a lot about iterating through and flattening data structures. An earlier version of the code naively filtered twice, and I _still_ think there's room for more filtering improvement (I think the dual for-loops is unnecessary). However, Lambda does not have a Ruby runtime, so this was an excellent opportunity to learn something new. 

I also learned how to use Serverless, which re-enforced my love for maintaining scripts to interact with your host (AWS in this instance). Scripting things allows me to reference them in the future instead of having to remember the steps I went through in the web console to make a Lambda event do what I wanted it to. 

_Thanks to my pals [Will](https://twitter.com/williamgfoster) and [Brik](https://twitter.com/_minikori) for help with this solution._





