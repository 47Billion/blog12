title: Delayed Jobs with Jesque
tags:
  - Delayed Job
  - Delayed Jobs
  - Java
  - Jesque
  - Quartz
  - Queue
  - Redis
  - Resque
  - Task Queue
id: 2002
categories:
  - Technology
date: 2013-09-02 17:06:43
---

[Jesque](https://github.com/gresrun/jesque "Jesque") is an interesting project. It's a Java port of [Github's Resque](https://github.com/resque/resque "Resque") task queue library. Works on top of Redis. It is fast. It eats less. And doesn't need a lot of maintenance, as long long your Redis is okay, you are okay! We have been happily using this in production for a while.

## Problem

And then, we encountered a need for delayed jobs. We needed an ability to execute jobs in future with deterministic delay. We had 2 options:

1\. Either, schedule these delayed jobs in [Quartz](http://quartz-scheduler.org/ "Quartz") - that was okay because we already had been using Quartz - and then, once Quartz jobs get fired, they publish a Task into Jesque, and let the workers handle the rest. (<span style="text-decoration:underline;">This was too contrived to implement, and would have become maintenance/reporting nightmare!</span>)

2\. We extend Jesque to support delayed jobs inherently.

## Solution

We decided to go with option-2, and started exploring Redis datasets. Turned out, [ZSET](http://redis.io/commands#sorted_set) was all that we needed.

Jesque uses Redis [LIST](http://redis.io/commands#list) for job storage, workers keep polling the list, and [LPOP](http://redis.io/commands/lpop)s tasks from the LIST.

This is what we ended up doing:

When <u>adding a delayed job</u>,

1\. Calculate future timestamp when the job should run,
2\. Use that timestamp as SCORE to ZSET entry.

```java
final long delay = 10; //sec
final long future = System.currentTimeMillis() + (delay * 1000); // future
jedis.zadd(QUEUE, future, jobInfo); 

// Redis
// ZADD &lt;queue&gt; &lt;future&gt; &lt;job-information&gt;
```

On the other hand, <u>Workers' poll logic was updated</u>. For Delayed queues, 

1\. Check if there are any items with SCORE between -INF and now,

```java
final long now = System.currentTimeMillis();
final Set&lt;String&gt; tasks = jedis.zrangeByScore(QUEUE, -1, now, 0, 1);
```

If tasks are non-empty, try to grab one to execute. 

```java
if (null != tasks &amp;&amp; !tasks.isEmpty()) {
    String task = tasks.iterator().next();
    // try to acquire this task
    if (jedis.zrem(QUEUE, task) == 1) {
         return task; // Return
    }
}
```

This way, we ensure that no 2 workers would grab the same task to execute. 

Also, an important point to note here is that - You <u>**don't have to change your existing workers**</u> or redo new workers in any way. Just bind them to a Delayed Queue, and start publishing delayed tasks. 

## Example

```java
// DelayedJobTest.java
package net.greghaines.jesque;

import static net.greghaines.jesque.utils.JesqueUtils.entry;
import static net.greghaines.jesque.utils.JesqueUtils.map;

import java.util.Arrays;

import net.greghaines.jesque.client.Client;
import net.greghaines.jesque.client.ClientPoolImpl;
import net.greghaines.jesque.worker.Worker;
import net.greghaines.jesque.worker.WorkerImpl;
import redis.clients.jedis.JedisPool;

public class DelayedJobTest {

    @SuppressWarnings(&quot;unchecked&quot;)
    public static void main(String[] args) throws InterruptedException {
        // Queue name
        final String QUEUE = &quot;fooDelayed&quot;;

        // Config
        final Config config = new ConfigBuilder().withHost(&quot;localhost&quot;).withPort(6379).withDatabase(0).build();

        // Client
        final Client client = new ClientPoolImpl(config, new JedisPool(&quot;localhost&quot;));
        long delay = 10; // seconds
        long future = System.currentTimeMillis() + (delay * 1000); // Future timestamp

        // Enqueue job
        client.delayedEnqueue(QUEUE, 
                new Job(
                        TestJob.class.getSimpleName(), 
                        new Object[] {&quot;HELLO&quot;, &quot;WORLD&quot; } // arguments
                ), 
                future);
        // End
        client.end();

        // Worker
        final Worker worker = new WorkerImpl(config, Arrays.asList(QUEUE), map(entry(TestJob.class.getSimpleName(), TestJob.class)));
        final Thread workerThread = new Thread(worker);
        workerThread.start(); // start
    }
}
```
And, this is the TestJob,

```java
// TestJob.java
package net.greghaines.jesque;

import java.util.Date;

public class TestJob implements Runnable {
    final String arg1;
    final String arg2;

    public TestJob(String arg1, String arg2) {
        this.arg1 = arg1;
        this.arg2 = arg2;
    }

    @Override
    public void run() {
        System.out.println(&quot;Job ran at=&quot; + new Date() + &quot;, with arg1=&quot; + arg1 + &quot; and arg2=&quot; + arg2);
    }
}
```

We have using this solution in production for quite sometime, and this has been pretty stable. 

BTW, you can grab the source here: [https://github.com/anismiles/jesque](https://github.com/anismiles/jesque) and give it a try yourself. 

Happy Hacking!