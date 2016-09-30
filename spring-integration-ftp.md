# Spring Integration - Add failover support to FTP inbound channel

## Problem description

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

To make better customization we need to know details of how exactly ``AbstractInboundFileSynchronizingMessageSource.receive()`` method works.
What we can find in javadoc for this method:

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

Ok, now we understand when and how files appear in our channel. Now let's talk what exactly is being accepted and then passed to processing.
In case of FTP we have **two independent filters**: filter for remote files and filter for local files.
There is a bunch of filters that could be used here. Let's look at the couple of them.

Following diagram illustrates interactions between parties involved into polling process:

![Sequence diagram](https://www.lucidchart.com/publicSegments/view/c7a64226-54f5-4ea3-a974-16b8802acaf1/image.png)

The root interface for all filters is ``FileListFilter`` with, again, only one method:

```java
/**
 * Filters out files and returns the files that are left in a list, or an
 * empty list when a null is passed in.
 *
 * @param files The files.
 * @return The filtered files.
 */
List<F> filterFiles(F[] files);
```

Pretty simple - take an array and return filtered list.

> Btw, it's kind interesting fact that it doesn't use List for both input and output.
> After digging sources I still don't understand why :(

Following diagram shows most interested filters: 

![Filter Hierarchy](https://www.lucidchart.com/publicSegments/view/8814916b-d80d-4c94-a5b7-4db9c82e457b/image.png)

These are used when we need to store some information about accepted files *between* polls in order to apply "accept once" semantics.
Interfaces ``ResettableFileListFilter`` and ``ReversibleFileListFilter`` contain methods used for removing information about files from store.

Most dummy implementation of "accept once" filter is ``AcceptOnceFileListFilter``. It use standard java ``Set`` structure to keep information about accepted files.
It also could be configured to keep only last N files in memory and after file falls out from queue it could be passed through this filter again.

> **NOTE:** Documentation says that it could be used both for ``FileReadingMessageSource`` and ``FtpInboundFileSynchronizingMessageSource`` as most simple to configure filter. 
> Unfortunately, it's not true due to implementation details. Filter stores accepted files in ``HashSet`` that is ok when you pass a class that implements ``hashCode()`` and ``equals()``.
> That is true for standard ``File`` and not true for ``FTPFile`` that is used by FTP message source.

Next implementation of "accepte once" is ``AbstractPersistentAcceptOnceFileListFilter`` that use ``ConcurrentMetadataStore`` for storing information about "seen" files.
This is abstract class and has own extension for File, FTP and SFTP cases.

Other filters that are worth to mention:

- ``IgnoreHiddenFileListFilter`` - filters all hidden files (e.g. file with name starting with dot in Linux) 
- ``RegexPatternFileListFilter`` - filters file list using regexp
- ``SimplePatternFileListFilter`` - filters file list using Ant like patterns
- ``LastModifiedFileListFilter`` - keeps files with lastmodified timestamp older than configured. One of the ways to filter files that are still being uploading.  
- ``AcceptAllFileListFilter`` - simple one that keeps everything
- ``HeadDirectoryScanner.HeadFilter`` - keeps only first N files from a list. **Note: should not be used in conjunction with an ``AcceptOnceFileListFilter`` or similar.**
- ``NioFileLocker`` - locks file before processing by using java.nio capabilities. Works good only with local filesystem.



## To be: Filter description, some diagrams

![State Diagram](https://www.lucidchart.com/publicSegments/view/c250b2de-9d91-4228-8267-448c106cf7c9/image.png)


## Demo



## limitations:
- file naming
- rejected channel


##Sources: main parts of implementation + link to full code and demo--


[si-ftp](http://docs.spring.io/spring-integration/reference/html/ftp.html)
[si-files](http://docs.spring.io/spring-integration/reference/html/files.html)











