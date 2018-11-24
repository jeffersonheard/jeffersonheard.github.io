---
layout: post
title:  "Region Cache - Russian Doll caching in Redis with Python"
date:   2018-11-24 10:58:41 -0500
categories: python libraries
---

> *Link to [region_cache](https://github.com/jheard-tw/region_cache) on GitHub. Also available on PyPi. Install via `pip install region_cache`*

It's been awhile since I've written anything in this blog, and I think it's time to change that. I've spent the last year and a half at [Teamworks](https://www.teamworks.com) as Principal Software Developer and Lead Architect for the Python backend.

Our data is highly hierarchical. For caching, we use Elasticache, which is AWS's [redis](https://redis.io) offering. I wanted to build a cache on top of that which worked well with the hierarchy we have, was easy-to-use from Python, and I wanted to solve the cache contention problems we were starting to run into. The result is the [region_cache](https://github.com/jheard-tw/region_cache) library. `region_cache` is a nesting-doll style cache that supports:

* Read-replicas.
* Read timeout.
* Cache timeouts per region.
* Dynamic reconnection.
* Pluggable serializers for values.
* Flushing of nested region when the outer region is flushed.
* Flushing of a cache region upon receiving a signal.
* Being used as a Flask extension.

Its only dependencies are [hiredis](https://github.com/redis/hiredis), [boltons](https://boltons.readthedocs.io/en/latest/), and [blinker](https://pythonhosted.org/blinker/). `region_cache` is meant to be used as a Flask extension, but it is not limited to that.

## So what problem was I trying to solve? 

### Logical nesting

The data we store is very often nested in a tree structure. If a node higher in the tree changes, it can affect the validity of the nodes below it, so if I change that, I would like to invalidate the cache both for the outer node and for all its children. Thus the nested-doll approach.  Nested caches are set up by either asking for dot-separated regions or by calling `.region(name, ...)` on a region.

{% highlight python %}
# either this
r = region_cache.region('abc.xyz')

# or this
r = region_cache.region('abc').region('xyz')
{% endhighlight %}

Now if you invalidate `abc`, you will also invalidate `xyz`. 

{% highlight python %}
r = region_cache.region('abc')
r.invalidate()
len(r.region('xyz'))
>>> 0
{% endhighlight %}

### High-level, Pythonic use

By default, the `pickle` serializer is used to store cache values. This is, of course, a security problem if anything can store data to your cache which is not a trusted part of your application. In that case, choose a different serializer. Serializers can be set on a per-region basis, so if you have certain regions that are untrusted, you could switch to json or yaml or something like that for serialization. Because the process of serialization and deserialization can be quite slow, the region_cache library keeps an LRU cache of deserialized values. As long as the serialized value is the same as the key stored in the LRU cache, the already deserialized value is used.

The `Region` class is a dictionary-like object supporting all the usual dictionary interfaces efficiently.

Additionally, it is also a context-manager. This allows you to write a series of values to the cache as a part of a single transaction like so: 

{% highlight python %}
with region as r:
    r['key1'] = 0
    r['key2'] = 1
{% endhighlight %}

### Contention

During peak times, we have a lot of contention for accessing our Elasticache. Using a read-replica is not at all straightforward with Flask-Cache or most high level libraries in Python for using redis. 

By default, the Python redis library's connection pool is unlimited. If a connection object is in-use, it simply makes another one. When connection counts get high, redis performance suffers and the whole application can suffer if you can't account for it. So the first thing I added to `region_cache` after nesting was a way to make the cache considerate by supporting timeouts and dropping of connections as soon as a timeout occurs.

Here is the doc for RegionCache's constructor params, which outlines what options are available to combat cache contention:

* `root` (optional str): Default 'root' The key to use for the base region.
* `serializer` (optional pickle-like object): Default = pickle. Flask/Celery config is
    `REGION_CACHE_SERIALIZER`.
* `host` (optional str): Default localhost The hostname of the redis master instance. Flask/Celery config is
    `REGION_CACHE_HOST`.
* `port` (int): Default 6379. The port of the redis master instance. Flask/Celery config is
    `REGION_CACHE_PORT`.
* `db` (int): Default 0. The db number to use on the redis master instance. Flask/Celery config is
    `REGION_CACHE_DB`.
* `password` (optional int): The password to use for the redis master instance. Flask/Celery config is
    `REGION_CACHE_PASSWORD`.
* `op_timeout` (optional number): Default = no timeout. A timeout in seconds after which an operation will
    fail. Flask/Celery config is `REGION_CACHE_OP_TIMEOUT`.
* `reconnect_on_timeout` (optional bool): Default = False. Whether to close the connection and reconnect on
    timeout. Flask/Celery config is `REGION_CACHE_OP_TIMEOUT_RECONNECT`.
* `reconnect_backoff` (optional int): Seconds that we should wait before trying to reconnect to the cache.
* `raise_on_timeout` (optional bool): Default = False. If false, we catch the exception and return None for
    readonly operations.Otherwise raise redis.TimeoutError. Flask/Celery config is
    `REGION_CACHE_OP_TIMEOUT_RAISE`.
* `rr_host` (optional str): Default None. The host for a redis read-replica, if it exists. Flask/Celery
    config is `REGION_CACHE_RR_HOST`.
* `rr_port` (optional int): The port for a redis read-replica.  MUST be set explicitly if using a read-replica. Flask/Celery config is `REGION_CACHE_RR_PORT`.
* `rr_password` (str): The password for the redis read replica. Flask/Celery config is
    `REGION_CACHE_RR_PASSWORD`.
* `args`: Arguments to pass to StrictRedis. Flask/Celery config is `REGION_CACHE_REDIS_ARGS`.
* `kwargs`: Extra options to pass to StrictRedis. Flask/Celery config is `REGION_CACHE_REDIS_OPTIONS`.

At its most polite, RegionCache will drop all connections as soon as it hits a timeout, flushing its connection pool and handing resources back to the Redis server. Then it will back off and use the local LRU cache for a predetermined time (`reconnect_backoff`) until it can connect to redis again.   