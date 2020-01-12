---
title: ECS for Self-Hosted Services
date: 2020-01-11 22:57:37
tags:
  - aws
  - self-hosted
---

For those who are unaware (I was until fairly recently), ECS is Amazon's "Elastic Container Service".
Basically, Docker containers as a service. With ECS, you have two options regarding the machines
where your containers will run : AWS Fargate or EC2. With Fargate, Amazon manages everything except the
container; you tell AWS what container you want to run and how much computing resources it should have,
and that's it, everything else is managed for you. This seems like a very appealing option for those of
us that like to host some services on their own.
