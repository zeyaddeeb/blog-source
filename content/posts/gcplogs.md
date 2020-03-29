---
date: 2020-03-29
title: "Google Cloud logs for Humans"
markup: mmark
---

I was an avid AWS user until a year ago, when I started using GCP for various Machine Learning projects.

Diving deeper into the clouds, I soon learned that logging becomes crucial to understand what is happening within your cloud environment.

In AWS, I used to go to the console to view my logs and make sure nothing was breaking, until one of my colleagues introduced me to [awslogs](https://github.com/jorgebastida/awslogs). This became my de facto "real-time" monitoring tool when I deploy/update a resource. I used it mainly to check if everything was smooth sailing. Later logs will hit whatever monitoring system you have in place, such as [prometheus](https://prometheus.io/), [elk stack](https://www.elastic.co/what-is/elk-stack), or whichever.

When I started using Google Cloud, I was irritated with going to the console and writing a query to view my logs, until I started debugging native VPCs for Kuberenetes, at which point I began looking for a tool like `awslogs` but unfortunately didn't find one. So, I built one `gcplogs` . You can find the source code on [GitHub](https://github.com/zeyaddeeb/gcplogs)

The tool is build using python, and you can easily install it using `pip` 

``` bash
pip install gcplogs
```

A sample usage:

``` bash
gcplogs get gce_instance --event-start='1 week ago'
```

{{% image "/images/gcplogs.gif" %}}

The most critical feature in this package is the ability to watch events as they happen using the `--watch` flag

``` bash
gcplogs get gce_instance --event-start='1 min ago' --watch
```

If you have been working in cloud infrastructure for a while, then you might know tools like [Terraform](https://www.terraform.io/), [Pulumi](https://www.pulumi.com/), and others. This is a preferred way to manage how resources are provisioned, to understand how resources interact with each other, see your access management, and much more. `gcplogs` is also a good tool to monitor how these tools are deploying resources to your environment.

I'm still working on unit tests and making the tool more robust, but early feedback is always much appreciated.

