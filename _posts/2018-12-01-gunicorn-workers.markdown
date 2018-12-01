---
layout: post
title:  "Pitfalls in Configuring GUnicorn for Production"
date:   2018-12-01 09:35:00 -0500
categories: python web
---

*(Tl;dr - don't use the `gevent` worker type unless you know all the libraries your applications uses and how they implement their I/O.)* 

I wish I could say that I was the one who found this, but it was our lead DevOps guy Andrew who did. For months, we've been plagued by random spikes in I/O wait times and occasional high-latency in serving API requests. The events would last for a few seconds, but if they hit at peak time they would back up the servers until we did a rolling restart of workers. 

We had a few options:

1. Badly formed requests from people with superuser access were causing really massive queries to be issued against our system.
2. We had a memory leak
3. We had contention for a resource
4. There was something wrong with our setup

Following the path down investigating these the first one was fairly easy to eliminate, because one bad query wouldn't block others and there's no way it'd drive up mean response time as much as what we were seeing. Sure they happen, but not as frequently or as severely as what we were seeing. 

I got stuck on the combination of having a memory leak and contention for a resource as "comorbid" problems. I have seen memory leaks before in Python (that is, dangling hard references to objects that prevent garbage collection). I've even seen them in Flask applications, usually to do with stuff in loaded Flask extensions. I even knew that one particular extension that I wrote in my early days at my job had some really ugly properties with regards to memory, so I resolved to clean that up. 

The path of debugging memory usage in Python is a hard path to tread. Without production-level loads and a good clean copy of production data one can mangle beyond recognition consequence-free, finding leaks is a lot of guesswork. But I knew where to start. I reworked the offending extension and got the memory WAY down and made it nicer to program against in the process. Satisfied, I wrote a bunch of tickets to have people port their code over from the old extension to the new one.

But while people were porting things a new symptom cropped up: one of our monitoring packages was showing ridiculous times to access O(1) operations on Redis. There's just no way that 400ms is a reasonable time to check TTL, do a BLPOP, or call EXPIRE. At first it looked as if there was some reporting artifact in the monitoring system. Maybe something in the way we set that up. But it happened consistently, and even after reconfiguring the monitoring package a bit it didn't go away. 

There was only one explanation, really, in my mind - resource contention (I was wrong as should be clear by the title of my post not being "Solving resource contention problems with Redis in Python"). What made this hypothesis so tantalizing?

* We knew of at least one legacy process that was really impolite with its use of redis, spamming the server with possibly tens of thousands of non-pipelined writes to different keys all at once. 
* We knew that legacy process was being used when at least some of the spikes occurred.
* It turns out that the default connection pooling method on Python's redis library is "unlimited," causing connection count to spike whenever one of these problems happened.  
* This connection count spike causes memory usage to spike both on the app-server side and on the redis server side (found our memory leak!)

The truly maddening thing is that *addressing the problem as a resource contention problem really did alleviate the symptoms*. As modules got ported over to [Region Cache]({% post_url 2018-11-24-region-cache %}) the API endpoints in those modules accounted for fewer of the spikes. Region Cache does a couple of things to "behave well" in high load scenarios: 

* Uses a socket timeout to make calls to operations fail on high redis load.
* Flushes its connection pool on socket timeout, returning resources to the redis server (and reducing memory footprint on its own side).

But resource contention was a symptom, not the cause. **The cause was our use of C extensions for accessing redis and rabbitmq in combination with our usage of the `gevent` worker type with gunicorn.**

Upon first read of the documentation on [gunicorn](http://docs.gunicorn.org/en/latest/configure.html), it looked like the `gevent` worker was our best choice. It monkey-patches I/O, making a cooperative multithreading system out of a worker. Our system is very I/O intensive. Very very few operations really do much with the CPU at all. Some of our more complicated operations may hit several backend services in the course of a single API call. A worker that would pass control along while one of these went into I/O wait seemed like the right choice. We went with it.

There is not easy to find documentation out there that tells you that C-extensions cause problems with `gevent`. At some point, we went from `amqplib` to `librabbitmq` and from `redis` in pure python to `hiredis` and therein was the source of all our problems. Andrew found it in a blog post that I can't even find anymore (I'll update this post with it when I remember to ask him). As soon as we knew what to look for, it was obvious, and there was plenty of documentation about `gevent.monkey_patch` that supported the "C extensions for I/O suck with gevent" hypothesis. 

Sure enough, switching to a different configuration with the default `sync` workers and HAProxy for queuing up incoming connections worked *beautifully* under load. We ran our load test suite against a sandbox of backend services on the old config vs. the new config. Once we got the stress high enough, we saw the same kinds of issues crop up on the old config. Spiky, latent I/O, high connection counts, and abysmal memory usage when hitting endpoints using Flask-Cache instead of  Region Cache.  

However when we switched to the new config the issues went away on all of those endpoints magically. We managed to hit our sandbox with 33% more users than we were hitting it with under the old config before response times noticeably increased. 

**Lessons Learned**: Start with the `sync` worker no matter your application. Switch to the `gevent` worker only after doing extensive load-testing or a complete evaluation of your library usage (or both) to see if it actually fits your use-case. 