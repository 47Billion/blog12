title: Jesque-Guice binding
tags:
  - Guice
  - Java
  - Quartz
  - Redis
  - Resque
id: 2031
categories:
  - Technology
date: 2013-09-18 16:44:41
---

In our Java stack, we use [Guice](https://code.google.com/p/google-guice/) IOC library quite heavily. We are kind of smitten with it. Now, when we needed a task queue to process our background jobs, we settled with [Jesque](https://github.com/gresrun/jesque) which is a port of Github's Resque project, and runs on Redis which was already part of our stack. In the process, we also added [delayed task functionality](http://anismiles.wordpress.com/2013/09/02/delayed-jobs-with-jesque/) to this. 

## Problem

Anyways, the problem we were facing was with Injecting Guice dependencies into Jesque workers. Our workers performed heavy operations and at times had to read-write from DB, or speak with other services. 

Jesque uses Java reflection APIs to instantiate and run its workers (which are Runnable classes) and so, it becomes very difficult to inject Guice managed objects and services into Jesque workers. 

We started the hacky-and-ugly way by creating static references to relevant services that our workers needed. if it were for few services, this would have worked, but our workers kept growing in features and reach, and after a while the whole code base was stinking. We had to do something about it. Something elegant!  

## Solution

So, we created Jesque Guice binding project. You can annotate your worker classes, and Guice will then discover, register and start them. Let me show you some code.

First, let's create an ExampleWorker. You have to remember that the Worker must <u>implement Runnable</u> interface, and it must be annotated with <u>@Worker</u> which Guice uses for discovery and binding. 
```java
// ExampleWorker

@Worker(job = &quot;ExampleJob&quot;,                 // job name
        queues = { &quot;EXP_QUEUE&quot; },           // queue names
        enabled = true,                     // enabled
        count = 1,                          // 1 instance of this worker running
        events = { WorkerEvent.JOB_SUCCESS, WorkerEvent.WORKER_START }, // Events to listen to
        listener = EchoListener.class      // WorkerEventListener
)
public class ExampleWorker implements Runnable {
    // LOG
    private static final Logger LOG = LoggerFactory.getLogger(ExampleWorker.class);

    // Note: Only field level injection would work!
    @Inject
    ExampleService service;

    String arg1;
    String arg2;

    // Must keep an empty constructor for Guice to discover this
    public ExampleWorker() {
    }

    public ExampleWorker(String arg1, String arg2) {
        this.arg1 = arg1;
        this.arg2 = arg2;
    }

    @Override
    public void run() {
        LOG.info(&quot;Running worker={}, with arg1={}, arg2={}&quot;, new Object[] { getClass(), arg1, arg2 });
        // calling service
        service.serve();
    }

}
```

**<u>@Worker</u>** attributes let you control the behavior. The above worker, ExampleWorker, listens to **<u>EXP_QUEUE</u>** queue, accepts Jobs by name **<u>ExampleJob</u>**, has an WorkerEventListener defined by Guice Managed class **<u>EchoListener</u>** which listens to WorkerEvent **JOB_SUCCESS** and **WORKER_START**. There is only **ONE** instance of this worker running. 

Note: ExampleWorker has been <u>field @Injected</u> with **ExampleService**. Please remember, that 
- <u>Only field @Inject</u> will work with Workers, because Jesque uses constructors to pass Job arguments. 
- Also, you <u>must keep an empty constructor</u> around so as to let Guice discover this worker. 

Now, since we have added a Listener, we must define it. 
```java
// EchoListener

public class EchoListener implements WorkerListener {
    // LOG
    private static final Logger LOG = LoggerFactory.getLogger(EchoListener.class);

    @Override
    public void onEvent(WorkerEvent event, Worker worker, String queue, Job job, Object runner,
            Object result, Exception ex) {
        LOG.info(&quot;onEvent ==&gt;&gt; queue={}, event={}&quot;, new Object[] { queue, event });
    }

}
```

For the sake of demonstration, this has been kept very minimal. But, mind you, you can @Inject any Guice managed objects into this, using <u>constructor and/or field injection</u>.  

Now, let's define the ExampleService that we want to @Inject into our worker. 

```java
// ExampleService

public class ExampleService {
    /** The Constant LOG. */
    private static final Logger LOG = LoggerFactory.getLogger(ExampleService.class);

    public void serve() {
        LOG.info(&quot;Heya! I am not here to serve, just to show you how injection works.&quot;);
    }
}
```

Wonderful! Let's now bind these all together in a GuiceModule. 

```java
// ExampleModule

public class ExampleModule extends AbstractModule {

    @Override
    protected void configure() {
        // Jesque Guice
        install(new JesqueModule());

        // Jesque Client
        Config config = new ConfigBuilder().withHost(&quot;localhost&quot;).withPort(6379).withDatabase(0).build();
        bind(Config.class).toInstance(config);
        bind(Client.class).toInstance(new ClientImpl(config));

        // Worker
        bind(ExampleWorker.class).asEagerSingleton(); // Must be singleton
        // WorkerEventListener
        bind(EchoListener.class).in(Scopes.SINGLETON);
        // Worker Executor (This is where they actually run)
        bind(WorkerExecutor.class).to(SimpleThreadBasedWorkerExecutor.class);
        // Service (will be injected into workers)
        bind(ExampleService.class).asEagerSingleton();
    }

}
```

Here, first we install JesqueModule, and then bind other objects. 
- Bind Jesque config and client.
- Bind Worker, Lister, Service etc. 

You will notice we have also bound <u>WorkerExecutor</u>. This interface accepts **<u>net.greghaines.jesque.worker.Worker</u>** instance and runs that on a thread. Jesque-Guice comes with 2 simple implementations:

1\. <u>SimpleThreadBasedWorkerExecutor</u> which run each net.greghaines.jesque.worker.Worker on an unmanaged separate thread, and
2\. <u>CachedThreadPoolBasedWorkerExecutor</u> which creates a CachedThreadPool where net.greghaines.jesque.worker.Worker is run. 

You can implement your own strategy or provide your own ExecutorService. 

## Run

Now we have everything, let's run it then. 

```java
// Main

public class Main {
    /** The Constant LOG. */
    private static final Logger LOG = LoggerFactory.getLogger(Main.class);

    /**
     * @param args
     */
    public static void main(String[] args) {
        Injector injector = Guice.createInjector(Stage.DEVELOPMENT, new Module[] { new ExampleModule() });

        // Get Jesque client
        Client client = (Client) injector.getInstance(Client.class);
        LOG.info(&quot;Publish jobs&quot;);
        // Push jobs
        client.enqueue(&quot;EXP_QUEUE&quot;, new Job(&quot;ExampleJob&quot;, &quot;hello&quot;, &quot;job1&quot;));
        client.enqueue(&quot;EXP_QUEUE&quot;, new Job(&quot;ExampleJob&quot;, &quot;hello&quot;, &quot;job2&quot;));
    }
}
```

You should see,

```bash
// Job - 1
DEBUG c.s.commons.jesque.GuiceAwareWorker - Injecting dependencies into worker instance = com.strumsoft.commons.jesque.example.ExampleWorker@6b754699
DEBUG c.s.commons.jesque.GuiceAwareWorker - Delegating to run worker instance = com.strumsoft.commons.jesque.example.ExampleWorker@6b754699
INFO  c.s.c.jesque.example.ExampleWorker - Running worker=class com.strumsoft.commons.jesque.example.ExampleWorker, with arg1=hello, arg2=job1
INFO  c.s.c.jesque.example.ExampleService - Heya! I am not here to serve, just to show you how injection works.
INFO  c.s.c.jesque.example.EchoListener - onEvent ==&gt;&gt; queue=EXP_QUEUE, event=JOB_SUCCESS

// Job -2
DEBUG c.s.commons.jesque.GuiceAwareWorker - Injecting dependencies into worker instance = com.strumsoft.commons.jesque.example.ExampleWorker@6602e323
DEBUG c.s.commons.jesque.GuiceAwareWorker - Delegating to run worker instance = com.strumsoft.commons.jesque.example.ExampleWorker@6602e323
INFO  c.s.c.jesque.example.ExampleWorker - Running worker=class com.strumsoft.commons.jesque.example.ExampleWorker, with arg1=hello, arg2=job2
INFO  c.s.c.jesque.example.ExampleService - Heya! I am not here to serve, just to show you how injection works.
INFO  c.s.c.jesque.example.EchoListener - onEvent ==&gt;&gt; queue=EXP_QUEUE, event=JOB_SUCCESS
```

Works... eh? :)

You can grab the project source at github [https://github.com/anismiles/jesque-guice](https://github.com/anismiles/jesque-guice/) Give it a try. I hope this will help some of you folks. Share your thoughts with me. 

Happy hacking!