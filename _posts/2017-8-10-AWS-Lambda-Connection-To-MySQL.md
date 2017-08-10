---
layout: post
title: AWS Lambda Connection to MySQL
post_author: Nicolás Kipreos
post_gravatar: 78ceb574aa61c77bf01f999ca64aebc9
---

![AWS Lambda](https://s3-us-west-2.amazonaws.com/assets.blog.serverless.com/AWS-Lambda.png){:height="200px"}

Each day that passes we are adopting more and more serverless apps for things such as integrations with our clients or monitoring some critical services that we use, for example, [Sidekiq](http://sidekiq.org/) queues or sms notifications. I can say that we are very happy with [AWS Lambda](https://aws.amazon.com/lambda/), but it still gives us a hard time once in a while.

For example, on the last function that we created we needed to connect to our MySQL read replica, located in [AWS RDS](https://aws.amazon.com/rds/) and send an email in case we ran into issues with the integration between [Beetrack](https://www.beetrack.com) and our SMS providers. If everything was ok we just needed to post the daily status of the sent messages into a [Slack](https://slack.com/) channel.

So, in summary, on one hand we needed the connection between Lambda and the RDS instance to be secure and highly restricted, but on the other hand we needed to connect with services like [AWS SES](https://aws.amazon.com/ses/) and Slack, so we couldn't locate the function in a VPC with a locally restricted connection to the replica, because that would have blocked the function's internet connection.

Fortunately, we had a VPC set up for Lambda functions that need a static IP address (I'll get into the details of that [story](#bonus-track-assigning-a-static-ip-to-lambda) later), so all what had to be done was to give inbound access to the replica in it's security group to that specific IP address, as shown in the following image.

![Read Replica Security Group Setup](/images/rr-security-group.png)

Once the conditions for connecting to the database where met, all I had to do was to was connect to it in my Python script, piece of cake, right?... WRONG, Lambda had other plans.

As I usually do I downloaded the needed library to my project's directory running the following commands:

```
pip install mysql -t .
pip install requests -t .
```

After compressing the files inside the directory as usual and uploading to lambda's console, I was getting my victory face ready, but...

![Kanye's Smile the Frown](https://meetedgar.com/wp-content/uploads/2016/07/Kanye-Frown.gif)

```
Unable to import module 'lambda_function': /var/task/_mysql.so: invalid ELF header
```

After doing some research I noticed that mysql's Python module needed some libraries that were compiled in an environment similar to that of the Lambda function, and I was uploading a Python module that was compiled in OS X. So, I just did all the work that I had done before but this time I did it in an EC2 instance with Ubuntu. After I connected to the instance, I basically ran the following commands:

```
mkdir your_project
cd your_project
pip install mysql -t .
pip install requests -t .
```

After that, I just compressed the files and uploaded them again, to my surprise, another error showd up:

```
Unable to import module 'lambda_function': libmysqlclient.so.18: cannot open shared object file: No such file or directory
```

This one was a little bit trickier, and got me researching a bit longer. Turns out that when you upload your files to lambda it doesn't only need the python modules to be on the directory but it also needs the system libraries to be in it. I solved this by copying the files into the directory (don't move them, just copy them!):

```
cd your_project
cp /usr/lib/x86_64-linux-gnu/libmysqlclient.so .
cp /usr/lib/x86_64-linux-gnu/libmysqlclient.so.18 .
cp /usr/lib/x86_64-linux-gnu/libmysqlclient.so.18.0.0 .
zip -r Archive.zip *
```

After that I just uploaded the zip file and it worked like a charm.

# Bonus Track: Assigning a Static IP to Lambda

Not so long ago I went to a meeting with an outsourcing IT company that was developing a middleware integration for a client. The meeting was to discuss some issues that we had with an integration. The problem was that the company's firewall was blocking our requests because they were not originated from a static know IP address, but from multiple IP's that changed with time (our web application uses dynamic IPs, and our servers reboot on a daily basis). After this was brought up in the meeting, they looked amazed and shocked, as if they never heard before of these "magical changing IPs". Obviously, after that reaction it was clear to see that the ability of them developing a more secure application rather than blocking that middleware to the world, but one IP, was null. So it was up to us to develop a middleware between [Beetrack](https://www.beetrack.com) and them, with a static IP. After the meeting with these guys ended, I went back to the office asking myself if their dev team looked something like this:

![Old School Dev Team](/images/old_school_dev_team.jpg)

I wanted this middleware integration to be a Lambda function so in order to do that I needed to arrange the function to run inside a VPC and communicate with the exterior world through a NAT Gateway. The following steps allowed me to achieve that and therefore, the static IP:

1. Create a VPC with a local 192.168.0.0/16 CIDR block as its local network.

2. Create 2 subnets, one public subnet with the 192.168.1.0/24 CIDR that attaches to the internet gateway through its route table, and a private subnet with the 192.168.2.0/24 CIDR block.

3. Create a NAT Gateway that attaches to the private subnet and to a static IP.

4. Assign your VPC to your Lambda function and on the subnets option, assign only the private subnet.

If you follow these steps, voila! you'll have a working Lambda function which performs all requests to the internet through the same IP address.