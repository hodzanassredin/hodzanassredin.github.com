--- 
published: true 
layout: post 
title: Using redis as a task queue 
tags : [redis, python, queue] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">06 April 2016 &#8211; Karelia</p> 
 

At work, some time ago we used azure storage queues as a task queue engine for our workers, but after some time we found that it is not as fast as we want it to be. At the same time, we used redis intensively and started to think about possibility to use it as a task queue. There is a lot of documentation how to use redis as publish/subscibe service, but not as much as a task queue. 
So I decided to describe how to do it.  
 
# What is a task queue? 
Task queue allows clients of some service asynchronously send tasks to it. Usually service has many clients and probably many workers. In short whole workflow looks like this:

1. Client puts task into a queue
2. Workers in loop periodically check the queue for a new task, if task exists then worker execute it

But there are some additional requirements to a queue:

1. Quality of service: One client should not block other client’s requests
2. Batch processing: clients and workers should have possibility to put and get several tasks at once for better performance.
3. Reliability: if worker fails during processing of a task after some time this task have to be handed by other or the same worker again.
4. Dead letters: if some task causes worker fail after several attempts to handle it then it have to be put into dead letters storage.
5. One task have to be processed only one time in case of success. 

# Implementation
We will use a redis list for every client. List key will use a task queue name as a prefix and second part will be client id. When client wants to put a message into the queue it will calculate list key by concatenation of the queue name and its own id. It would be great to put these lists into a separate redis database, but in case it we have to share redis db with some other code lets add an additional prefix like "queues:" to their names. So let’s define a class RedisQueue which will hide implementation details.

{% highlight python %} 
import json
import datetime
import pytz
from random import randint
import logging
import time

main_prefix = "bqueues:"

class ClientRedisQueue():
    def __init__(self, qname, clientid, redis):
        self.client_id = clientid
        self.queue_name = main_prefix + qname + ":" + clientid
        logging.debug("created queue %s" % self.queue_name)
        self.redis = redis

    def send(self, msgs):
        jmsgs = [json.dumps({ "client_id" : self.client_id, "msg" : msg, "attempts" : 0}) for msg in msgs]
        self.redis.lpush(self.queue_name, *jmsgs)

    def exists(self):
        return self.redis.exists(self.queue_name)

    def length(self):
        return self.redis.llen(self.queue_name) 

    def delete(self):
        self.redis.delete(self.queue_name)

r = redis.StrictRedis("localhost")
cq = ClientRedisQueue("worker1", "client", r)

cq2 = ClientRedisQueue("worker1", "client2", r)
cq.send([1,2])
cq2.send([3,4,0])
{% endhighlight %} 
So sending was easy to implement and what about receiving side?
First of all, we need to find all queue lists. There are three options:


1. Use KEYS "prefix:*" command for all lists search. But this command could cause serious problems on production, it may ruin performance when it is executed against large databases. So never use this option.
2. use SCAN command this command works the same as keys, but has no performance problems.
3. Store all names in a redis set and add a list name to a set on massage sending and delete a queue form a set when all messages are processed. Unfortunately, this step requires additional code to implement so we will use second option.

When we found all queues, we need to randomly sort them to guaranty that all queues have the same probability to be processed. After that, we need to get required count of messages in one batch (with redis pipeline). After that if no messages found then we need to run whole process again in other case handle messages and delete them after processing. Also we need to prevent double processing of a message in a list and prevent message lose caused by exceptions during message processing, to do that we will use RPOPLPUSH command which atomically removes message from a list and put it into an additional "processing" list and return value to a caller. So we will use additional list for every queue with key "processing:queue_name". After message handling we must remove it from prccessing list. But in case of several unsuccessful attempts to process message we need to finally move it to a deed letters. As a dead letters store we will use redis set with key "dead:queue_name". From time to time we need to check processing list and if attempts count of a message is lower than max allowed count then put it back to a client list or in other case put it into dead letters set.

{% highlight python %} 
MAX_ATTEMPTS_COUNT = 4
class WorkerRedisQueue():
    def __init__(self, name, redis):
        self.queue_name = main_prefix + name
        self.processing_lname = main_prefix + "processing:" + name
        self.dead_sname = main_prefix + "dead:" + name
        self.refresh_period = datetime.timedelta(seconds=2)
        self.queue_pattern = self.queue_name + "*"
        self.redis = redis
        self.last_refresh_time = datetime.datetime.now(pytz.utc) - self.refresh_period - self.refresh_period
        self.list_names = []

    def proccessed(self, msg):
        self.redis.lrem(self.processing_lname, 0, json.dumps(msg))

    # start this from time to time
    def recover(self):
        recovered = 0
        proc_msgs = self.redis.lrange(self.processing_lname, -5,-1)
        for (msg, orig) in [(json.loads(msg),msg) for msg in proc_msgs if msg]:
            if msg["attempts"] > MAX_ATTEMPTS_COUNT:
                print "found dead letter"
                self.redis.sadd(self.dead_sname, orig)
            else:
                print "recovering"
                recovered = recovered + 1
                msg["attempts"] = msg["attempts"] + 1
                self.redis.lpush("%s:%s" % (self.queue_name, msg["client_id"]), json.dumps(msg))
            self.redis.lrem(self.processing_lname, 0, orig)

        return recovered

    def get_list_names(self):
        lists = []
        print "searching pattern", self.queue_pattern
        for l in self.redis.scan_iter(self.queue_pattern):
            print "found list", l
            lists.append(l)
        return lists

    def refresh(self, force = False):
        now = datetime.datetime.now(pytz.utc)
        time_to_refresh = now - self.last_refresh_time > self.refresh_period
        if force or time_to_refresh:
            self.list_names = self.get_list_names()
            self.last_refresh_time = now
        else:
            print "skip refresh"

    def receive(self, count):
        self.refresh()
        count = len(self.list_names)
        if count == 0:
            print "queues not found"
            return []
        p = self.redis.pipeline()
        for i in range(count):
            l = self.list_names[randint(0, count - 1)]
            p.rpoplpush(l,self.processing_lname)
        msgs = p.execute()
        return [json.loads(msg) for msg in msgs if msg]

    def length(self):
        self.refresh(True)
        res = 0
        for l in self.list_names:
            res = res + self.redis.llen(l)
        return res

wq = WorkerRedisQueue("worker1", r)
while(True):
    time.sleep(1)
    msgs = wq.receive(2)
    if len(msgs) == 0:
        if randint(0, 10) == 0 and wq.length() == 0 and wq.recover() == 0:
            print "sleeping"
            time.sleep(1)
            
    for msg in msgs:
        print "received msg", msg
        try:
            a = 10/msg["msg"]
            wq.proccessed(msg)
        except Exception,e: 
            print "exception", str(e)   

{% endhighlight %} 

[Full code](https://gist.github.com/hodzanassredin/d014d36ef9296c3343afcc4609c7b2fa)

# Recommended reading: 
1. [Redis’ reliable queue pattern](https://danielkokott.wordpress.com/2015/02/14/redis-reliable-queue-pattern/) 
2. [Redis Pipelines and Transactions](http://www.terminalstate.net/2011/05/redis-pipelines-and-transactions.html)
