# Spring Integration

Add failover support to FTP inbound channel



## Intro: Problem description

Imagine a situation when you have FTP server or SAN/NAS (mounted to local system) where all your clients put their's files and you import them into you database.
 Sounds easy, right? Actually, it becomes a hard pain when you have distributed environment and you'd like to make your import process reliable.

Situation that should be kept in mind while designing your solution:

- you may want to process your files concurrently
- file download from FTP to local filesystem may fail
- JVM may crash while processing downloaded file
- mostly you will not want files be processed more than once or ignored - usually we need exactly once semantics (or at least with a certain degree of confidence)

In our case we had geo distributed cluster, one FTP server, shared filesystem and requirements listed above.
![Our deployment schema](https://www.lucidchart.com/publicSegments/view/bbd534be-f3ad-4711-a11c-e2e3e1204fae/image.png)



## As is: describe in short what do we have already implemented in spring

Spring Integration has a lot of predefined channels adapters including [FTP/FTPS][si-ftp] and local [filesystem][si-files].
But after some digging in documentation and sources we can find that it's not enough for our case and we must implement some stuff by ourselves.

First let's look deeper into Spring Integration features.

### MessageSource

On the top of channels hierarchy lies ``MessageSource`` interface with one simple method:
```java
/**
 * Retrieve the next available message from this source.
 * Returns <code>null</code> if no message is available.
 * @return The message or null.
 */
Message<T> receive();
```

And when you use ``<int-ftp:inbound-channel-adapter />`` you actually configure a bean of ``FtpInboundFileSynchronizingMessageSource`` class.

![Spring FTP Message Source class diagram](https://www.lucidchart.com/publicSegments/view/8eb40b68-0bd6-4210-9fd1-f03aaf933138/image.png)

Class ``AbstractInboundFileSynchronizingMessageSource`` has two significant responsibilities:

- configure ``fileSource`` with appropriate filters
- orchestrate calls to ``synchronizer`` and ``fileSource``

To make better customization we need to know details of how exaclty ``AbstractInboundFileSynchronizingMessageSource.receive()`` method works.
What we can find in javadoc:

> Polls from the file source. If the result is not null, it will be returned.
> If the result is null, it attempts to sync up with the remote directory to populate the file source.
> Then, it polls the file source again and returns the result, whether or not it is null.

It means that each time we call ``recieve()`` it will do:

1. it tries to poll local directory for files (using ``fileSource``)
2. then if there is nothing in local directory it will try to synchronize remote folder with local one and poll it again

How often method ``recieve()`` is called depends on poller configuration (either it is global poller or a specific one for this channel).
Following example shows configuration when each 5sec method ``recieve()`` will be called twice:

```xml
<int-ftp:inbound-channel-adapter id="ftpInbound" ...>
    <int:poller fixed-rate="5000" max-messages-per-poll="2" />
</int-ftp:inbound-channel-adapter>
```

The rest of details could be found in source files.

### Filters

Ok, now we understand when and how files appear in our channel. Now let's talk what exactly is being processed.
In case of FTP we have two sets of filters. There is a bunch of filters that could be used here. Let's look through a couple of them.

*diagram with filters*





> Note: problems in documentation


## To be: Filter description, some diagrams

![State Diagram](https://www.lucidchart.com/publicSegments/view/c250b2de-9d91-4228-8267-448c106cf7c9/image.png)


## Demo



## limitations:
- file naming
- rejected channel
- 


##Sources: main parts of implementation + link to full code and demo--


[si-ftp](http://docs.spring.io/spring-integration/reference/html/ftp.html)
[si-files](http://docs.spring.io/spring-integration/reference/html/files.html)











