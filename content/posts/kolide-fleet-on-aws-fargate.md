---
title: Kolide Fleet on AWS Fargate
description: In which I waste time finding out how to do this 'AWS native', somehow
date: 2020-08-12T17:21:02+02:00
lastmod: 2021-02-11T17:21:02+02:00
tldr: I worked my way through a Kolide Fleet deployment on AWS Fargate a week ago, and have turned the results into a set of CloudFormation templates for your consumption. GitHub repo available <a href="https://github.com/chessmango/kolide-fleet-on-fargate">here</a>.
tags:
  - aws
  - container
  - fargate
  - git
  - iac
  - serverless
draft: false
---

[Kolide] seems like a cool company. I've never used their commercial offering, but Kolide Fleet[^fleet] is pretty and works well, and Kolide osquery Launcher[^launcher] is pretty great for [osquery] without much headache.

{{< callout text="This post is *not* sponsored by Kolide, promise" >}}

I've been playing around with _osquery_ for a while, though it's been very single-use for me, tinkering more than anything. Last week I had the good fortune of assisting a customer who was attempting to make use of Kolide Fleet on Fargate[^fargate] (using ElastiCache for Redis[^redis], Aurora for MySQL[^mysql], Elastic Load Balancing[^elb] and encryption at rest ~~everywhere~~ wherever posssible), which is a _simple_ matter except for all the caveats.

\
I set things up according to [bare necessity] and had good success, but there's one fairly large catch: **no Application Load Balancer**. One of Kolide Launcher's selling points is [gRPC], which uses HTTP/2 for transport. While AWS ALBs **do** support HTTP/2, the traffic is distributed as individual HTTP/1.1 requests[^http1], which means Kolide itself needs to handle TLS if you want to keep HTTP/2 and gRPC alive (hint: you probably do).

So we use a Network Load Balancer. How do we handle TLS? Unfortunately, that's where things get iffy. Amazon-issued certificates are for AWS services only, so you get your certificates ([acme.sh] is my poison, with [Let's Encrypt] as provider) and either:

- Copy the certs as part of your container build
  - Rebuild your container image every time you replace a cert
  - Perhaps this isn't that big a deal

**OR**

- Import certs into [ACM] and pull them from the container directly
  - Cool, easy cert replacement
  - Feels nice because it's all cloudy
  - But your private key can't be pulled from ACM
  - Gotta use [SSM Parameter Store] for private key

You could arguably just use Parameter Store for all the moving parts, but ACM is nicer because it tells you when things expire ¯\\\_\(ツ\)\_/¯

\
So seeing as we're cloud-crazy (?) and want to make all of this serverless at every single point, let's go for Let's Encrypt certs (Route 53 DNS plugin in my case even, why not) imported into ACM, with private key stored as a [SecureString] SSM parameter for later retrieval. We'll encrypt everything we can (at rest) with [KMS] and generally use as many AWS services as possible.

Without further ado, enter a [handy set] of sample [CloudFormation] templates that'll get you going, as well as a [`Dockerfile`] and modified [`run.sh`] for Kolide Fleet's container. The template order is quite intentional, infrastructure first:

- [`01-ecr.yml`]
  - Creates an ECR repository, 'nuff said
  - Use the outputs from this (including a handy dandy `ecr get-login-password | docker login` snippet!) to push your container image
- [`02-kms.yml`]
  - Creates a KMS key for:
    - SSL certificate private key stored as SecureString SSM parameter
    - RDS Aurora master password, stored in [Secrets Manager]
    - RDS Aurora cluster encryption at rest
    - ElastiCache Redis cluster encryption at rest
  - Manually use the KMS ID to encrypt a new SecureString SSM parameter for your SSL certificate private key, and note the parameter name
  - At the same time, import your cert, private key and full chain into ACM, note the certificate ARN
- [`03-vpc.yml`]
  - Creates a VPC with:
    - CIDR 10.0.0.0/16
    - 1x public and 1x private subnet and routes in AZ a and b for your region
  - The earlier the better - there's a lot to provision here :D
- [`04-security-groups.yml`]
  - Creates security groups for:
    - Kolide service on Fargate
    - ElastiCache for Redis
    - Aurora for MySQL
  - Depends on [`03-vpc.yml`]
- [`05-rds-aurora.yml`]
  - Generates a master password and stores it with Secrets Manager
  - Creates an RDS Aurora MySQL cluster
- [`06-redis.yml`]
  - Creates an ElastiCache Redis cluster
- [`07-network-load-balancer.yml`]
  - Creates a Network Load Balancer for use with the Kolide service
  - Outputs a FQDN, for targeting wherever you keep your DNS
- [`kolide.yml`]
  - Creates suitable IAM roles and policies for Kolide tasks and orchestration
  - Creates an ECS task definition along with two container definitions
    - One container redirects port 80 to 443 with Nginx
    - The other container is Kolide Fleet, and gets environment variables according to all the fancy resources defined above

\
Once done, you're done! This may take some time - I have some further plans for testing and seamlessness, namely:

- Investigate using TLS termination at the NLB, potentially (?) retaining HTTP/2
  - This would mean being able to use Amazon-issued public certs, and therefore avoiding awkward ACM imports and SSM parameters
- Create a meta-template that spawns all the stacks involved and saves a bunch of manual work
  - Even better if terminating TLS at the NLB - no more human dependency

\
That's it for now! Give that [repo] a watch - I'm likely to push a second branch at some point after testing TLS on NLB, along with some instructions away from this blog, probably. Let me know what you think in the comments below! And feel free to file an issue on GitHub if you see one ;)


<!-- Links -->
[Kolide]: https://www.kolide.com/
[osquery]: https://osquery.io/

[bare necessity]: https://github.com/kolide/fleet/blob/master/docs/infrastructure/installing-fleet.md#infrastructure-dependencies
[gRPC]: https://grpc.io/

[acme.sh]: https://github.com/acmesh-official/acme.sh
[Let's Encrypt]: https://letsencrypt.org/

[ACM]: https://aws.amazon.com/certificate-manager/
[SSM Parameter Store]: https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html

[SecureString]: https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-about-examples.html#parameter-types
[KMS]: https://aws.amazon.com/kms/

[handy set]: https://github.com/chessmango/kolide-fleet-on-fargate
[CloudFormation]: https://aws.amazon.com/cloudformation/
[`Dockerfile`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/app/docker/Dockerfile
[`run.sh`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/app/docker/run.sh

[`01-ecr.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/01-ecr.yml
[`02-kms.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/02-kms.yml
[Secrets Manager]: https://aws.amazon.com/secrets-manager/
[`03-vpc.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/03-vpc.yml
[`04-security-groups.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/04-security-groups.yml
[`05-rds-aurora.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/05-rds-aurora.yml
[`06-redis.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/06-redis.yml
[`07-network-load-balancer.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/infrastructure/07-network-load-balancer.yml
[`kolide.yml`]: https://github.com/chessmango/kolide-fleet-on-fargate/blob/master/app/kolide.yml

[repo]: https://github.com/chessmango/kolide-fleet-on-fargate


<!-- Footnotes -->
[^fleet]: https://www.kolide.com/fleet/
[^launcher]: https://www.kolide.com/launcher

[^fargate]: https://aws.amazon.com/fargate/
[^redis]: https://aws.amazon.com/elasticache/redis/
[^mysql]: https://aws.amazon.com/rds/aurora/
[^elb]: https://aws.amazon.com/elasticloadbalancing/

[^http1]: https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-listeners.html#listener-configuration
