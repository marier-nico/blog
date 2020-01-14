---
title: ECS for Self-Hosted Services
date: 2020-01-11 22:57:37
tags:
  - aws
  - self-hosted
---

## What is ECS?

For those who are unaware (I was until fairly recently), ECS is Amazon's "Elastic Container Service". Basically, Docker containers as a service, meaning you only manage docker containers and not the machined which run them. With ECS, you have two options regarding the machines where your containers will run : AWS Fargate or EC2. With Fargate, Amazon manages everything except the container; you tell AWS what container you want to run and how much computing resources it should have, and that's it, everything else is managed for you. This seems like a very appealing option for those of us that like to host some services on their own, but there is one important problem with this solution. As you may have guessed, pricing is the main concern here.

## ECS on Fargate

Before going any further, I should mention that the pricing examples below will be using the standard pricing (i.e. on-demand). Any exceptions will be mentioned.

AWS advertises Fargate as being a "pay for what you actually use" service, but it's not quite that simple. In fact, when creating your service with AWS Fargate, you are asked what resources to allocate to your container. So far so good, select something low-powered enough for the costs to make sense and off you go, right? Not really. I was hoping for something like that when I first tried Fargate, but obviously, there's a well-defined minimum you can't go under. For Fargate, pricing information can be found [here](https://aws.amazon.com/fargate/pricing/). It seems fine at first : "$0.04048 per vCPU per hour" and "$0.004445 per GB [of memory] per hour" (at the time of writing). That works out to be around $33.4242 per month for one vCPU and one GB of memory, assuming the worst-case of 31 days in a month. Not too bad if the services that are hosted on Fargate bring in some sort of revenue, but for personal use, that's a bit steep. Now you might be thinking we can just lower the specs and get a good deal, I thought about that too, but the minimum configuration is 0.25 vCPU and 0.5 GB, which translates to $9.18282 per month.

### Fargate on the cheap?

Stretching things to the extreme (let's say we **really** want Fargate), it's possible to use Amazon's [Savings Plans](https://aws.amazon.com/savingsplans/pricing/), where resources are not on-demand, but reserved ahead of time. Depending on the length of time the resource is reserved for and how much is paid upfront, it's possible to get a discount up to 52% on the regular Fargate price. Seems like an amazing deal, right? Well, it still turns out to be around $4.4077536 per month with the smallest configuration possible, a three year term and a full upfront payment. That's actually quite a good price except when you consider that that's the price for **one** task or pod within Fargate. If you have, say, a self-hosted [Heimdall](https://heimdall.site/) dashboard and a [BookStack](https://www.bookstackapp.com/) wiki, that's two tasks, so about $9 per month. For someone that wants to host many services in the cloud for reliability or speed, that's going to stack up very quickly.

Perhaps that's worth it to you if the hassle of managing machines really _is_ unacceptable, but for services that often only have a single user, I just can't justify that price. Do understand that I'm not knocking on Amazon for their pricing, it might make a lot of sense for a different workload, but for personal self-hosted services, it just doesn't work for me.

## ECS on EC2

Now, I was saying at the beginning that Fargate wasn't the only option, it is also possible to run ECS-managed containers on EC2 instances. After finding out that Fargate pricing wasn't for me, I thought maybe it would still be possible to benefit from ECS while using (much cheaper) EC2 instances, because ECS can be used at no extra cost over the EC2 instance(s) used to run the containers. EC2 pricing varies grately depending on the instance type and size. Because requirements are exceedingly low for most self-hosted applications, the EC2 instance discussed here will be the `t3a.nano`, which offers 2 vCPUs and 0.5 GiB of memory, which is the same amount of memory as the smallest Fargate configuration, but it has 8x more vCPUs. Pricing for that instance can be found [here](https://aws.amazon.com/ec2/pricing/on-demand/). It will cost about \$3.4968 per month, which is already lower than Fargate's Savings Plans pricing. It can get even lower than that with [reserved instances](https://aws.amazon.com/ec2/pricing/reserved-instances/pricing/) as well, which only costs \$1.488 per month on a three year plan and the full payment upfront. That seems like a very reasonable price for small, self-hosted services to me.

### Caveats with ECS on EC2

Since ECS adds no extra cost over what the EC2 instance costs, surely it's worth it to use ECS to manage containers, then! Well, not so fast. First of all, I should mention that workloads in ECS are separated into "tasks", which are themselves made up of container configurations. Now, since all the containers are running on the one EC2 instance (unless you fancy getting more), exposing services can be tricky. There are many networking options for tasks, but the most interesting one is "host" networking. Essentially, any ports open by container configurations inside a task will be directly accessible when connecting to the EC2 instance's public IP. The first problem I ran into was wanting to run more than one web service in different tasks. As you probably know, it's not really possible to expose ports 80 and 443 twice, which led me to try to use a reverse proxy, like I would normally do in my docker-compose configuration.

Exposing the reverse proxy to the Internet using host networking was very straightforward, but the trickier part was to get the reverse proxy to talk to other services. What I didn't know at the start was that tasks are isolated from each other completely, which makes sense, but still adds a barrier to successfully using ECS. Essentially, the configuration I was trying to reproduce as a proof of concept was the following :

```yml
version: "3"

services:
  heimdall:
    image: linuxserver/heimdall
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - ./heimdall:/config

  caddy:
    image: abiosoft/caddy
    restart: unless-stopped
    volumes:
      - ./caddy/Caddyfile:/etc/Caddyfile:ro
      - caddycerts:/root/.caddy
    ports:
      - 80:80
      - 443:443
    environment:
      ACME_AGREE: "true"

volumes:
  caddycerts:
```

#### Task networking with ECS on EC2

The easy way to do this would be to just create one task with all the containers inside it, but that's a bit of an anti-pattern and it doesn't really seem much better than just using docker-compose on an EC2 instance anyway, so I ruled out that possibility (but you _could_ do that, it's an option). The real way to this is much less straight forward, and a lot more expensive than the EC2 alone. The way to do it is to use [task networking with the `awsvpc` task network mode](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html). This network mode provides each task with its own Elastic Network Interface, which give ECS tasks the same networking properties as EC2 instances. This network mode seems like it solves all our problems and, in theory, it does. Although, in practice, there is a limit to how many ENIs an EC2 instance can have attached, which depends on the instance type. For the `t3a.nano`, the [limit](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI) is 2 and one of them is already used up by the default interface, which leaves room for only one extra task that uses the `awsvpc` network mode.

That might seem like a deal-breaker already, because running only one task plus one reverse proxy on an entire EC2 instance is definitely overkill (and you might as well just run the task on its own with host networking since it'll be the only one), but Amazon has considered this limitation and allows the use of [ENI trunking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-eni.html), which can raise the limits of ENIs on a single EC2 instance. Unfortunately, the only [supported](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-eni.html#eni-trunking-supported-instance-types) instance type families are : a1, c5, m5, p3 and r5. So it seems this is not a viable solution for the `t3a.nano` instance type which would be the most cost-effective.

## Conclusion

What I'm sticking with, personally, is probably the simplest solution but it's quite effective and most definitely the easiest to setup. For my own needs, a single `t3a.nano` instance is plenty, and I use docker-compose to manage my various services. The downsides to this are that updates have to be applied manually on the EC2 instance (though, the AMI used for ECS over EC2 also needs manual updates) and that there are no convenient integrations with other AWS services, like having the ability to load environment variable values to pass to containers from AWS Parameter Store. Those are not too dramatic for me, so overall, this whole deep dive into ECS was more of a learning experience than anything else.
