---
layout: post
title: redis & sentinel thoughts after some doc reading
image: /img/redis-sentinel.png
tags: [redis,sentinel]
---

Recently i had to start reading some sentinel documentation due to some migrations being conducted at my work place. I never had the chance to do it and always relied on the 'documentation' existing in config files and what i read in company communication channels. That is always wrong but you know, kids, daily tasks among others, stopped me to read this [redis-sentinel documentation](https://redis.io/topics/sentinel) and hence, getting more beter knowledge/input around the features, behaviour etc..

I'm no going to go deep. Will just make some notes of what i consider interesting facts around redis sentinel.

#### What is redis sentinel

> Redis Sentinel provides high availability for Redis.
> In practical terms this means that using Sentinel you can create a Redis deployment that resists without human intervention to certain kind of failures.

#### What is redis sentinel NOT used for

- Sentinel doesn't manage connections initiated by clients.  

#### What is redis sentinel used for

- Monitoring
- Alerting
- Redis Discovery
- Automate failover

**Monitoring and alerting** are basics, these allow us to monitor and using the sentinel api, to grab information regarding the status of redis instances.
**Redis Discovery or also called Service Discovery** let us use Redis Sentinel to discover config like what instance is master/slave and drive requests through them. This can be used by redis client with sentinel support for managing a HA scenario from app prespective.
**Automatic Failover** will do what the word says, failover a master to promote a slave to become master.

#### Achieving HA Redis

Sentinel will cope with it by tying itself against the redis instances to be monitored. Sentinel wil be managing slave / master instances and will act upon then health on them. There are several paramenters that can be configured to control how and when instances are failed over.

Sentinel instances monitor by themselves a set of redis instaces. The data that is retrieved by doing that monitoring is following:

- Which redis instance is master/slave
- Which sentinel server has got attached to.
- Redis instance status, like down, up etc.

All  Sentinel instances toghether (3 of them is ideal) must  achieve a quorum to take decisions, so with the data previously gathered, they communicate each other (sentinels) and vote for a decision and given that the number of sentinels  should be odd, there  should be no problem unless complex network partitions are taking place.

There is no config in place to let the sentinel instances to comm each other but that data is gathered from redis.

#### Achieving HA Redis from app prespective

In order to achieve HA is not enough with setting up Sentinel along with redis to manage the cluster properly. We gonna need to think of best way to drive requests to the right instance, for example, slaves for reading and master for writes. (Bear in mind syncronization among redis instances is async so read could be returning inconsistent/no up to date information)

| By mean of                     | Refer to |
| :---                           |   ----:  |
| Using Redis Client             | [Redis Clients](https://redis.io/clients)|
| Using alternative Ha methods   | [HaProxy](http://www.haproxy.org/)|


One example of **redis client** would be StackExchange.Redis. This will allow us have a highly available and meeting all required perfomance needs. Using multiplexer funcionalities allows us to pass a set of servers from which a master will be populated. This is not using redis Sentinel itself.

There are other clients that support usage of **Sentinel** to identify the master where connection should be made against. One example i could be thinking of is to use consul service discovery to hit sentinel and return the master instance where consul ready applications could be querying to.

Another way to set up a HA redis infra without using Sentinel itself would be using **Haproxy** where the there would be some rules for the haproxy backends  that would execute prechecks to deliver the  connectiong to the right server.

<pre>
  option tcp-check
  tcp-check connect
  tcp-check send AUTH\ password\r\n
  tcp-check expect string +OK
  tcp-check send PING\r\n
  tcp-check expect string +PONG
  tcp-check send info\ replication\r\n
  tcp-check expect string role:master
  tcp-check send QUIT\r\n
  tcp-check expect string +OK
  server host1 1.1.1.1:6379 check
  server host2 1.1.2.1:6379 check
  server host3 1.1.3.1:6379 check
</pre>

#### Sentinel and Redis facts

- **min-slaves-to-write** can be used to avoid critical situations when network paritions take place.  Clients could be writting to old masters, this would prevent it.
- Sentinel, Docker, or other forms of **Network Address Translation** or Port Mapping should be mixed with care
- In Sentinel, only masters are specified to be monitored, each one with a different name. Slaves will be self-discovered
- Sentinel **will not forget sentinel servers** in case they are decommissioned. They need to be forgotten by trigerring RESET command. Not resetting the snetinel config is useful, because Sentinels should be able to correctly reconfigure a returning slave after a network partition or a failure event. If not that should be done to have the right state of sentinel servers
- **SDOWN** sentinel status is Subjective Down and it is tied to a sentinel instance. On the contrary **ODOWN** means Objective Down which is status achieved by quorum of all sentinel instances.
- **Sentinel configuration is persisted to disk** so restarting the sentinel instance is safe to do so when restarting will pick up the most recent config in the files.

#### Sentinel and Redis useful commands

``` bash
redis-cli -p 26379 SENTINEL failover $redisInstance
```

>This will promote a slave of the current master as master and master will become slave of this one. This is achieved by internall calling redis  instance and issuing **slaveof** and **slaveof no one** commands

```bash
redis-cli -p 26379 SENTINEL reset $redisInstance/*
```

>This will reset redis sentinel known config and will be populated again within less than 10 seconds. Safe to do and required when old sentinel instances been decommisioned

```bash
redis-cli -p 26379  SENTINEL masters
```

> Show a list of monitored masters and their state

```bash
redis-cli -p 26379  slaveof $redisIsntace
```

> make a redis instance to be slave of given redis instance

```bash
redis-cli -p 26379  slaveof no one
```

> make a redis instance to be master
