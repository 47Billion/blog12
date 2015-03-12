title: Jetty Executable Webserver Archetype
tags:
  - Deployable
  - Embedded Jetty
  - Google Guice
  - Guice
  - Java
  - Jetty
  - REST
  - Resteasy
  - self-executing-war
id: 1979
categories:
  - Technology
date: 2013-01-08 13:48:49
---

We create and deploy many rest based API apps on our servers. Now, instead of managing multitudes of webservers on different machines, we make our deployable archives wrapped with all the necessary dependencies in order to be self-executable. Benefits?

*   We just have to ensure that all our servers have the same Java environment, and no other dependencies.
*   No waste of time configuring, managing web (tomcat, jetty etc.) servers.
*   And, we only have to distribute a single executable.
Hassle free management. Won’t you agree?

To cut down on bootstrapping time, we have created a [maven archetype](http://maven.apache.org/guides/introduction/introduction-to-archetypes.html) to help us quickly create and start on new API projects. Our tech stack is [Google Guice](http://code.google.com/p/google-guice/), [JBoss Resteasy](http://www.jboss.org/resteasy), [Logback](http://logback.qos.ch/), and [Embedded Jetty](http://www.eclipse.org/jetty/).

Github link: [https://github.com/anismiles/jetty-webserver-archetype](https://github.com/anismiles/jetty-webserver-archetype)

## Install

You can either build/install from source

```java
git clone https://github.com/anismiles/jetty-webserver-archetype.git
mvn clean install
```

Or you can simply [download the jar](https://github.com/anismiles/jetty-webserver-archetype/blob/master/downloads/jetty-webserver-archetype-1.0.jar?raw=true) and install that.

```java
mvn install:install-file \
-Dfile=jetty-webserver-archetype-1.0.jar \
-DgroupId=com.strumsoft \
-DartifactId=jetty-webserver-archetype \
-Dversion=1.0 \
-Dpackaging=jar \
-DgeneratePom=true
```

## Usage

Let’s say you want to create a new project “**hello-world**” with group “**com.hello.world**”, you would run:

```java
mvn archetype:generate \
-DgroupId=com.hello.world \
-DartifactId=hello-world \
-Dversion=1.0-SNAPSHOT \
-DarchetypeGroupId=com.strumsoft \
-DarchetypeVersion=1.0 \
-DarchetypeArtifactId=jetty-webserver-archetype
```

That’s it. You are ready to roll. You have set up a basic working API app. Now, you can run it in dev mode

```java
mvn clean jetty:run
```

Or in production mode

```java
mvn clean package
java –jar target/hello-world-1.0-SNAPSHOT-dist.war start &amp;
```

To check, open [http://localhost:8085/](http://localhost:8085/)</pre>
To stop:

```java
java –jar target/hello-world-1.0-SNAPSHOT-dist.war stop
```

Further, you can pass additional Jetty, Logback or your App's properties

```java
java \
-Djetty.configurationFile=&lt;jetty-config&gt; \
-Dapp.configurationFile=&lt;app.properties&gt; \
-Dlogback.configurationFile=&lt;logback.xml&gt; \
–jar target/hello-world-1.0-SNAPSHOT-dist.war start &amp;
```

They will override default properties setup by the executable.</pre>
Ideally you might want to create an [init.d](http://www.ghacks.net/2009/04/04/get-to-know-linux-the-etcinitd-directory/) file to start and stop you API app efficiently. Here is a prototype:

```bash
#!/bin/sh

# Executable war file
WAR_FILE=/var/apps/&lt;my-war-file&gt;.war

# Configuration files
APP_NAME=&lt;app-name&gt;

JETTY_CONFIG=/etc/${APP_NAME}-jetty.properties
APP_CONFIG=/etc/${APP_NAME}-app.properties
LOGBACK_CONFIG=/etc/${APP_NAME}-logback.properties

# Process
PIDFILE=/var/run/${APP_NAME}.pid

# Java arguments
JAVA_ARGS=&quot;-Djetty.configurationFile=${JETTY_CONFIG} -Dapp.configurationFile=${APP_CONFIG} -Dlogback.configurationFile=${LOGBACK_CONFIG}&quot;

# Command
CMD=$1

if [[ -z ${CMD} ]]; then
  java ${JAVA_ARGS} -jar ${WAR_FILE} usage
  exit 1
fi

if [[ ${CMD} == 'start' ]]; then
    if [[ -f ${PIDFILE} ]]; then
        echo &quot;Already running&quot;
        exit 1
    fi

    if [[ -f ${WAR_FILE} ]]; then
        echo &quot;Starting jetty: ${WAR_FILE}&quot;
        java ${JAVA_ARGS} -jar ${WAR_FILE} start &amp;
        PID=$!
        echo &quot;$PID&quot; &gt; ${PIDFILE}
        echo &quot;Started ${APP_NAME} with pid: ${PID}&quot;
    fi

elif [[ ${CMD} == 'stop' ]]; then
    # Try gracefully first
    java ${JAVA_ARGS} -jar ${WAR_FILE} stop
    sleep 10
    if [[ -f ${PIDFILE} ]]; then
        PID=`cat ${PIDFILE}`
        test -z $PID || kill $PID
        rm ${PIDFILE}
        sleep 1
        echo &quot;Forcibly Stopped ${WAR_FILE} with pid: ${PID}&quot;
    fi

else # Just let the other cmds through...
    java ${JAVA_ARGS} -jar ${WAR_FILE} ${CMD}
fi

exit 0
```

That’s it! have fun. :)