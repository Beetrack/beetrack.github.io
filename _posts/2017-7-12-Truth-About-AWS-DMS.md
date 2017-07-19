---
layout: post
title: The Truth About AWS DMS
post_author: Nicolás Kipreos
post_gravatar: 78ceb574aa61c77bf01f999ca64aebc9
---

![AWS DMS](https://pbs.twimg.com/media/CnRPccHUIAAdXrU.jpg)

At first when we heard of AWS [Managed Database Migration Service (DMS)](https://aws.amazon.com/es/dms/), we got pretty exited. The idea of a managed service that could migrate all of our data without having any downtime sounded too good to be true... and it was.

The need for doing the migration was to move our old MySQL database hosted in AWS [Relational Database Service (RDS)](https://aws.amazon.com/es/rds/), into another RDS instance, but locating it inside a [Virtual Private Cloud (VPC)](https://aws.amazon.com/es/vpc/), this way the database would be more secure because it would only be accessible to servers through a local network and it wouldn't have a direct access to the internet.

So, back to DMS. Things got complicated pretty quickly, because our first choice was selecting a small instance as shown in the following picture.

![Replication Instance](/images/replication_instance.png)

The downside about this is that apparently our databases where too big for the job, so the server ran out of memory and threw an error. That was solved by choosing a bigger instance in our case a medium one, we also set the number of tables to be loaded on parallel to 4 (default is 8) in the tasks advanced settings, in that way we minimized the load on the replication instance.

![Resplication Task Advanced Settings](/images/task_advanced_settings.png)

Another thing to consider is that unlike EC2 instances, which you only pay if their are running, you must pay for replication instances whether they are running or stopped, so watch out so you don't incur in overcharges for something you are not using.

All these thing were just a minor setback, the real deal happened after the database was completely migrated and the replication task was only updating and creating new records.

When we tried to run the application, things went south real quick. For apparently no reason, there were some validations that didn't work with our new database. After a long debugging session, we noticed that it had something to do with columns that were intended for Boolean values in our application. Further analysis of the table's structure, showed that columns that were created on the origin database as TINYINT(1) by RoR, where created as TINYINT(4) by the replication instance. This was our exact reaction when truth hit us:

![WTF](/images/wtf.gif)

From here we had only two options:

1. Run an ALTER TABLE command for each table to change TINYINT(4) columns to TINYINT(1):

![How About No](/images/how_about_no.jpg)

This option was discarded pretty quickly, mainly because, nothing assured us that, once that we made the changes we would face again with another mind-blowing discovery.

{:start="2"}
2. Take a snapshot of the production database and create another RDS instance from the snapshot:

This was the absolute winner, mainly because we are a B2B company that gives tools for other companies to enhance their operations which mainly occur during week days, so we could afford 15 minutes of downtime on a Sunday at 6:00 pm.

Unfortunately, if you can't afford to have any downtime I guess I would recommend you to continue with option number one to see what it happens. Another option would be doing option number 2 but while you are changing from one DB to another storing all incoming information in a queue that you would have to process once you are up and running. Anyway, I hope this is useful and serves as a heads up if your considering in using DMS.