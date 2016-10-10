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
![Our deployment schema](https://www.lucidchart.com/publicSegments/view/d9713ffd-0b6f-438e-ac88-4e1bb55a25be/image.png)



## As is: describe in short what do we have already implemented in spring

Spring Integration has a lot of predefined channels adapters including [FTP/FTPS](http://docs.spring.io/spring-integration/reference/html/ftp.html) and local [filesystem](http://docs.spring.io/spring-integration/reference/html/files.html).
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

![Spring FTP Message Source class diagram](https://www.lucidchart.com/publicSegments/view/da1455ee-6c98-4bd1-aec0-182c95d73d20/image.png)

Class ``AbstractInboundFileSynchronizingMessageSource`` has two significant responsibilities:

- configure ``fileSource`` with appropriate filters
- orchestrate calls to ``synchronizer`` and ``fileSource``

To make better customization we need to know details of how exactly ``AbstractInboundFileSynchronizingMessageSource.receive()`` method works.
What we can find in javadoc for this method:

> Polls from the file source. If the result is not null, it will be returned.
> If the result is null, it attempts to sync up with the remote directory to populate the file source.
> Then, it polls the file source again and returns the result, whether or not it is null.

It means that each time we call ``receive()`` it will do:

1. it tries to poll local directory for files (using ``fileSource``)
2. then if there is nothing in local directory it will try to synchronize remote folder with local one and poll it again

How often method ``receive()`` is called depends on poller configuration (either it is global poller or a specific one for this channel).
Following example shows configuration when each 5sec method ``receive()`` will be called twice:

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

![Sequence diagram](https://www.lucidchart.com/publicSegments/view/8f82a79c-1588-4cab-a9a0-291ca873bf69/image.png)

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

![Filter Hierarchy](https://www.lucidchart.com/publicSegments/view/d6a0ecf2-604f-418f-b55d-4b453f8fda53/image.png)

These are used when we need to store some information about accepted files *between* polls in order to apply "accept once" semantics.
Interfaces ``ResettableFileListFilter`` and ``ReversibleFileListFilter`` contain methods used for removing information about files from store.

Most dummy implementation of "accept once" filter is ``AcceptOnceFileListFilter``. It use standard java ``Set`` structure to keep information about accepted files.
It also could be configured to keep only last N files in memory and after file falls out from queue it could be passed through this filter again.

> **NOTE:** Documentation says that it could be used both for ``FileReadingMessageSource`` and ``FtpInboundFileSynchronizingMessageSource`` as most simple to configure filter. 
> Unfortunately, it's not true due to implementation details. Filter stores accepted files in ``HashSet`` that is ok when you pass a class that implements ``hashCode()`` and ``equals()``.
> That is true for standard ``File`` and not true for ``FTPFile`` that is used by FTP message source.

Next implementation of "accept once" is ``AbstractPersistentAcceptOnceFileListFilter`` that use ``ConcurrentMetadataStore`` for storing information about "seen" files.
This is abstract class and has own extension for File, FTP and SFTP cases.

[Documentation](http://docs.spring.io/spring-integration/reference/html/system-management-chapter.html#metadata-store) says about metadata store:

> The Metadata Store is designed to store various types of generic meta-data (e.g., published date of the last feed entry that has been processed) to help components such as the Feed adapter deal with duplicates.

Filter stores in a store information about accepted files in following format:
```
"file name" -> "last modified timestamp"
```

Timestamp is used for cases when we update file and it is passed to processing again.

Some failover capabilities are added to ``AbstractPersistentAcceptOnceFileListFilter`` filter. In case of any failure during synchronization (downloading) ``AbstractInboundFileSynchronizingMessageSource`` calls ``rollback()`` method that removes entries about files from metadata store.
 
Other filters that are worth to mention:

- ``IgnoreHiddenFileListFilter`` - filters all hidden files (e.g. file with name starting with dot in Linux) 
- ``RegexPatternFileListFilter`` - filters file list using regexp
- ``SimplePatternFileListFilter`` - filters file list using Ant like patterns
- ``LastModifiedFileListFilter`` - keeps files with lastmodified timestamp older than configured. One of the ways to filter files that are still being uploading.  
- ``AcceptAllFileListFilter`` - simple one that keeps everything
- ``HeadDirectoryScanner.HeadFilter`` - keeps only first N files from a list. **Note: should not be used in conjunction with an ``AcceptOnceFileListFilter`` or similar.**
- ``NioFileLocker`` - locks file before processing by using java.nio capabilities. Works good only with local filesystem.

## Why it is not enough?

There are lot of stuff already implemented in Spring Integration, though we can use it for rollback only when error happens in code but there is nothing for handling JVM crashes: files are marked as accepted *before* any processing starts. Also default implementations accept *all* files just after we get a list from FTP or Local file system - it makes concurrent processing almost impossible.

## Extending standard filters

We decided to implement our own filter that will not just accept files but also mark processing as completed once it is over (with configured retry timeout). Also we added ability to limit number of files to accept.

![State Diagram](https://www.lucidchart.com/publicSegments/view/dc269ce7-c439-443c-a24a-95621b01f118/image.png)

We store more information in metadata store then we need to serialize and deserialize some structure (e.g. JSON as in example):
```
"ftpLocalAcceptOnceRetriableFilter-c.txt" ->"{\"status\":1,\"tries\":1,\"lastTryTimestamp\":1474975591}"
```

Implementation of ``accept(F file)`` method is trivial:
```java
@Override
protected boolean accept(F file) {
    String key = buildKey(file);
    synchronized (monitor) {
        long currentTimestamp = Instant.now().getEpochSecond();

        FileAcceptStatus status = new FileAcceptStatus();
        status.setLastTryTimestamp(currentTimestamp);
        status.setTries(1);

        String newValue = StatusSerializer.toString(status);
        String oldValue = store.putIfAbsent(key, newValue); // try happy path

        if (oldValue != null) {
            do {
                oldValue = store.get(key);
                status = StatusSerializer.fromString(oldValue);

                if (status.getStatus() == FileAcceptStatus.DONE) {
                    return false;
                } else if ((currentTimestamp - status.getLastTryTimestamp()) < retryTimeoutSeconds) {
                    return false;
                }

                if (status.getTries() >= maxTry) {
                    status.setStatus(FileAcceptStatus.REJECTED);
                } else {
                    status.setLastTryTimestamp(currentTimestamp);
                    status.setTries(status.getTries() + 1);
                }

                newValue = StatusSerializer.toString(status);
            } while (!store.replace(key, oldValue, newValue));
        }

        return status.getStatus() == FileAcceptStatus.IN_PROGRESS;
    }
}
```
And ``commit(F f)``:
```java
@Override
public void commit(F file) {
    String key = buildKey(file);
    synchronized (monitor) {
        String oldValue;
        String newValue;

        do {
            oldValue = store.get(key);
            FileAcceptStatus status;

            if (oldValue != null) {
                status = StatusSerializer.fromString(oldValue);

                if (status.getStatus() == FileAcceptStatus.DONE) {
                    /*
                     * another process finished processing before our process - this should be reported
                     * with high severity and timeout value must be increased
                     */
                    return;
                }
            } else {
                // very strange situation when file was processed without creating record in metadata store
                status = new FileAcceptStatus();
            }

            status.setStatus(FileAcceptStatus.DONE);
            newValue = StatusSerializer.toString(status);
        } while (!store.replace(key, oldValue, newValue));
    }
}
```

Configuration of this filter looks like:
```xml
<bean name="ftpLocalAcceptOnceRetriableFilter" class="com.epam.cc.java.ftp.prototype.FilePersistentAcceptOnceRetriableFileListFilter">
    <constructor-arg ref="metadataStore"/>
    <constructor-arg value="ftpLocalAcceptOnceRetriableFilter-"/>
    <property name="retryTimeoutSeconds" value="60"/>
    <property name="maxAcceptedFileListLength" value="5"/>
</bean>
```

We have configured our filter and it will store data about accepted files. Now we need to configure call to ``commit`` method. Spring documentation offers so called ``transaction-synchronization-factory`` component that might be configured with three callbacks ``beforeCommit``, ``afterCommit`` and ``afterRollback``. It's a kind of confusing but we will configure it to call ``commit`` method in both ``afterCommit`` and ``afterRollback`` callbacks - this is because software errors must be handled in another way - we just ensure that file was processed nevertheless processing was successful or not.

```xml
<int-ftp:inbound-channel-adapter id="ftpInbound" ...>
    <int:poller fixed-rate="5000" max-messages-per-poll="2" task-executor="executor">
        <int:transactional synchronization-factory="syncFactory" />
    </int:poller>
</int-ftp:inbound-channel-adapter>

<int:transaction-synchronization-factory id="syncFactory">
    <int:after-commit expression="@ftpLocalAcceptOnceRetriableFilter.commit(payload)"  />
    <int:after-rollback expression="@ftpLocalAcceptOnceRetriableFilter.commit(payload)"  />
</int:transaction-synchronization-factory>
```
**Note:** we also must have transaction manager configured. It could be either ``PseudoTransactionManager`` or any other.

And we still no done yet. All this configuration is only for local filter. If we want to configure remote filter we must *hack* spring integration implementation of ``AbstractInboundFileSynchronizer`` to make it friendly to our implementation.

## Extending Default Synchronizer

As it usually happens when you want to customize Spring classes you need to read lot of source code, SO posts and forums. In our case we will write very simple extension for ``AbstractInboundFileSynchronizer`` and switch to Java configuration as most simple way to replace default implementation with the custom one.

```java
public class FtpExtendedInboundFileSynchronizer extends AbstractInboundFileSynchronizer<FTPFile> {

    private CommitableFilter<FTPFile> commitableFilter;

    ...

    @Override
    protected void copyFileToLocalDirectory(String remoteDirectoryPath, FTPFile remoteFile, File localDirectory,
                                            Session<FTPFile> session) throws IOException {
        super.copyFileToLocalDirectory(remoteDirectoryPath, remoteFile, localDirectory, session);
        if (commitableFilter != null) {
            commitableFilter.commit(remoteFile);
        }
    }

    public void setCommitableFilter(CommitableFilter<FTPFile> commitableFilter) {
        this.commitableFilter = commitableFilter;
    }

    ...
}
```
And it becomes interesting when you need to configure it :)

```java
    @Bean
    public AbstractInboundFileSynchronizer ftpInboundFileSynchronizer() {
        FtpExtendedInboundFileSynchronizer fileSynchronizer = new FtpExtendedInboundFileSynchronizer(ftpSessionFactory());
        fileSynchronizer.setDeleteRemoteFiles(deleteRemoteFiles);
        fileSynchronizer.setRemoteDirectory(remoteDirectory);
        **fileSynchronizer.setFilter(ftpRemoteCompositeFilter());**
        **fileSynchronizer.setCommitableFilter(ftpPersistentFilter());**
        return fileSynchronizer;
    }

    @Bean
    @InboundChannelAdapter(channel = "ftpInbound", poller = @Poller("poller"))
    public MessageSource<File> ftpMessageSource() {
        FtpInboundFileSynchronizingMessageSource source =
                new FtpInboundFileSynchronizingMessageSource(ftpInboundFileSynchronizer());
        source.setLocalDirectory(new File(localProcessingDirectory));
        source.setAutoCreateLocalDirectory(true);
        source.setLocalFilter(ftpLocalCompositeFilter());
        return source;
    }
```
You can't use DSL and configuration via XML won't be trivial as well.

## Demo Application
Sample project has simple integration flow: we download files from FTP, process them (fixed 10s for each) and then move files to directory with processed files:

![Demo flow](https://www.lucidchart.com/publicSegments/view/3ed3db2b-4b9e-4d84-8e9d-5293318c1972/image.png)

We configure processing to run in parallel by setting executor property of a poller.
To support Tx boundaries we use ``DirectChannel`` to ensure that all flow happens within single thread.

## Next steps



## limitations:
- file naming
- rejected channel

