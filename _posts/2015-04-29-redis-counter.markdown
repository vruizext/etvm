---
layout: post
title:  'Tracking counts with Python and Redis'
date:   2015-06-29
categories: python redis
---

  [Redis](http://redis.io) is an open source, high performance key-value in memory store similar to
[memcached](http://memcached.org/). The two major differences between them is, first, that redis can be persisted
on disk, whereas  memcached is volatile. And second, unlike memcached and other tradicional key-value stores which only
allow to store string values associated to string keys, redis is actually a data structures server, with support for
different kind of values besides plain strings, like lists, sets, sorted sets or hashes.

  In this post I'm going to show how to use [sorted sets](http://redis.io/commands#sorted_set) to track counts of a list
of events (visits per page, messages sent per user, page views per day, etc) in near real time. A typical use case would
be track count of the visits of every page in a site, which could be used later to build a list of the most popular articles.
As redis is thread-safe and ensures atomicity of the commands, it can be used as well in a distributed environment, i.e.,
even when we had a cluster of web servers, we could keep all page views in only one redis counter.

  The pages in the website will be the members, and the number of visits will be the score of every member in our sorted
set. For every page hit we simply store a page identifier and increment the number of visits. This problem could
be also solved with a hash structure, but the sorted set suits us better for this case because it's pre-sorted, i.e. the
members are taken sorted by their score and it's possible to retrieve a range of elements (for example, get the top 5).

### RedisHelper
  I created the `RedisHelper` class (`redishelper.py`) with some functions to handle redis connections. Using a connection
pool will allow to reuse the connections in the pool rather than create a new connection every time a command is sent to
redis. The pool is created the first time a connection is requested, and maintained as a class attribute. The get_pipe
method, returns a redis pipeline which can be used to send a batch of commands which will be executed automatically.
This can dramatically improve the performance, of groups of commands, since the traffic between the client and redis
server is reduced.

{% highlight python %}
import redis

class RedisHelper(object):

    server = {}

    @classmethod
    def set_server(cls, server):
        cls.server = server

    @classmethod
    def get_pool(cls):
        try:
            pool = cls.pool
        except AttributeError:
            pool = redis.ConnectionPool(host=cls.server['host'], port=cls.server['port'], db=cls.server['db'])
        return pool

    @classmethod
    def get_connection(cls):
        return redis.Redis(connection_pool=cls.get_pool())

    @classmethod
    def get_pipe(cls):
        return cls.get_connection().pipeline()

    @classmethod
    def flushdb(cls):
        cls.get_connection().flushdb()
{% endhighlight %}<br/>

### RedisZSetCounter
  Now let's see the implementation of the `RedisZSetCounter` class. In the constructor, we set the config for the redis
server and for the counter. the `prefix` is intended to identify a group of counters which measures the same thing. For
every counter in a group we'll have a `counter_id`. For example, if we might create a group of counters to measure page hits
per day, the keys would have the following format: `hits:21-06-2015`, `hits:22-06-2015`, etc., where 'hits' would be the
`prefix`, and the date would be the `counter_id`.

{% highlight python %}
class RedisZSetCounter(object):
    """
    Counter that uses redis to store data
    """
    def __init__(self, server, config):
        RedisHelper.set_server(server)
        self.redis = None
        self.prefix = config['prefix']
        self.ttl = config['ttl']

{% endhighlight %}<br/>

  The keys for every counter are built concatenating the prefix and the counter identifier.

{% highlight python %}
    def get_key(self, counter_id):
        """
        build the key that identifies a counter
        :return: counter key (string)
        """
        return "%s:%s" % (self.prefix, str(counter_id))
{% endhighlight %}<br/>

  Using the `RedisHelper` class we get a redis connection.

{% highlight python %}
    def get_redis(self):
        """
        Get redis connection, lazy loading
        :return: redis object
        """
        if self.redis is None:
            self.redis = RedisHelper.get_connection()

        return self.redis
{% endhighlight %}<br/>

  To increase the count of an entry, we use redis `zincrby` command. By default, we increase the score by one, but it's
possible to pass an arbitrary value.

{% highlight python %}
    def incr(self, counter_id, entry, count=1):
        """
        Increments the count of member_id in the current bucket by count
        Supports decreasing, when count < 0
        :param counter_id:  identifier for this counter
        :param entry: identifier of the member / item / whatever is being accounted
        :param count: increase count by this amount
        :return: the new count for this entry
        """
        key = self.get_key(counter_id)
        value = self.get_redis().zincrby(key, entry, count)
        self.get_redis().expires(key, self.ttl)
        return value

{% endhighlight %}<br/>

  It's also possible to pass a negative value to `zincrby`. In that case, the score is decreased by the specified amount.

{% highlight python %}
    def decr(self, counter_id, entry, count=1):
        """
        Decrease counter, by calling incr with negative count
        :param counter_id: suffix identifier for this counter
        :param entry: identifier of the member / item / whatever is being accounted
        :param count: decrease count by this amount
        :return: the new count of the entry
        """
        return self.incr(counter_id, entry, -count)
{% endhighlight %}<br/>

  The last N items can be retrieved using `zrange`. We need to specify the parameter `desc=False`, because we don't want
the entries in descending order but in increasing. Since dictionaries in python don't preserve the order of the keys, we
are using a list of tuples to handle the key-values returned by the counter.

{% highlight python %}
    def last_n(self, counter_id, how_many):
        """
        get the last N entries in this counter
        :param counter_id: identifier for this counter
        :param how_many: how many items to return (top N)
        :return: list of tuples (id, count)
        """
        key = self.get_key(counter_id)
        return self.get_redis().zrange(key, 0, how_many - 1, desc=False, withscores=True)
{% endhighlight %}<br/>

  Using `zrevrange` command, we retrieve the top N items with highest score. Note that `zrange` with `desc=True` would
also do the job here.
{% highlight python %}
    def top_n(self, counter_id, how_many):
        """
        get the top N entries in the counter
        :param counter_id: identifier for this counter
        :param how_many: how many items to return (top N)
        :return: list of tuples (id, count)
        """
        key = self.get_key(counter_id)
        return self.get_redis().zrevrange(key, 0, how_many - 1, withscores=True)
{% endhighlight %}<br/>

### How to use it

  Finally, let's see with an example how to use `RedisZSetCounter`
{% highlight python %}
    from redishelper import RedisHelper
    from redisZcounter import RedisZSetCounter

    redis_server = {'host': 'localhost', 'port': 6379, 'db': 0}
    params = {'prefix': "hits", 'ttl': 120}
    zcount = RedisZSetCounter(self.redis_server, params)
    RedisHelper.redis_flushdb()
    zcount.incr('test_id', 'page_1', 1)
    zcount.incr('test_id', 'page_2', 3)
    zcount.incr('test_id', 'page_3', 1)

    best = zcount.top_n('test_id', 1)
    # best => ('page_2', 3)

{% endhighlight %}<br/>


### Wrapping up

  Even though the use case presented in this post is very simple, we can see some of the advantages of using a noSQL database as
redis over a relational database to do real-time state management. Besides the simplicity and the performance, it's the power
 of its data structures. In my next posts I will extend the functionality of this counter and show how to use it to track
 event counts with time series.

  For those who are lazy to type, you can find the code in my github [repository](https://github.com/vruizext/RedisZCounter).
To run the code if you haven't yet, you'll need to install redis server and the redis client for [python](https://github.com/andymccurdy/redis-py).





