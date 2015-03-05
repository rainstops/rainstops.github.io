---
layout: post
title: DiskLruCahe 的实现
category: "Android"
---

上一篇介绍了使用 LruCache 缓存图片。LruCache 有比较明显的缺陷，那就是使用内存做缓存空间，会受到内存大小的限制。而且我们都知道这个限制比较严重 :-) 。

对于网络图片，仅仅使用 LruCache 是不够的，因为缓存在 LruCache 中的图片很可能过一段时间就被清理掉了。从网络下载图片又比较慢，而且耗费流量，这种情况下，DiskLruCache 就派上用场了。

------

### DiskLruCache 是什么

顾名思义，DiskLruCache 是使用磁盘（闪存）做缓存空间的 LruCache 。

------

### DiskLruCache 的作用

以下三种情况下使用 DiskLruCache 有助于提高图片加载速度：

>* 加载网络图片
>* 加载视频缩略图
>* 需要显示的图片较小，原图较大

第一点上面已经提到过。

第二点，一般从媒体数据库中取视频缩略图还是比较快的，但是在某些情况下，媒体数据库中的缩略图质量并不能满足要求，需要从视频中取一帧作为缩略图。这时候将取出的视频帧放入 DiskLruCache 中，下次直接从 cache 中取出视频帧会快一些。

第三点，在一些相册类应用中，会使用 ListView 或 GirdView 显示大量的图片缩略图，这些缩略图都有固定的显示大小。如果每次都直接原图中取缩略图，是比较慢的。在第一次从原图中取出缩略图后，将缩略图存入 DiskLruCache 中，下次直接从 cache 中取出缩略图，会快许多。

------

### 实现一个 DiskLruCache
实现一个简单 DiskLruCache，只需要磁盘空间和维护 LRU 顺序的数据结构。磁盘空间肯定是有的，维护 LRU 顺序的数据结构上一篇已经介绍过：LinkedHashMap。根据图片信息创建合适的 < Key, Value>，放入 LinkedHashMap，再将图片写入磁盘空间。需要图片时通过 LinkedHashMap 取出图片存放路径，读取出 Bitmap。

但是要实现一个完善的 DiskLruCache，不得不考虑以下问题：

>* 如何在程序重新启动时恢复缓存数据（包括 LRU 顺序）？
>* 如何避免缓存错误？

我们来看一下 Google 是如何实现的。在 Android 官网的 [Tranning](http://developer.android.com/training/displaying-bitmaps/index.html) 板块可以下载图片管理的 Demo：[DisplayingBitmaps](http://commondatastorage.googleapis.com/androiddevelopers/samples/DisplayingBitmaps.zip). Demo 中有一个 DiskLruCache 类。

------

### Google 的 DiskLruCache 类

下面是 DiskLruCache 内部的一段注释：

```
    /*
     * This cache uses a journal file named "journal". A typical journal file
     * looks like this:
     *     libcore.io.DiskLruCache
     *     1
     *     100
     *     2
     *
     *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
     *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
     *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
     *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
     *     DIRTY 1ab96a171faeeee38496d8b330771a7a
     *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
     *     READ 335c4c6028171cfddfbaae1a9c313c52
     *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
     *
     * The first five lines of the journal form its header. They are the
     * constant string "libcore.io.DiskLruCache", the disk cache's version,
     * the application's version, the value count, and a blank line.
     *
     * Each of the subsequent lines in the file is a record of the state of a
     * cache entry. Each line contains space-separated values: a state, a key,
     * and optional state-specific values.
     *   o DIRTY lines track that an entry is actively being created or updated.
     *     Every successful DIRTY action should be followed by a CLEAN or REMOVE
     *     action. DIRTY lines without a matching CLEAN or REMOVE indicate that
     *     temporary files may need to be deleted.
     *   o CLEAN lines track a cache entry that has been successfully published
     *     and may be read. A publish line is followed by the lengths of each of
     *     its values.
     *   o READ lines track accesses for LRU.
     *   o REMOVE lines track entries that have been deleted.
     *
     * The journal file is appended to as cache operations occur. The journal may
     * occasionally be compacted by dropping redundant lines. A temporary file named
     * "journal.tmp" will be used during compaction; that file should be deleted if
     * it exists when the cache is opened.
     */
```

上面这段注释介绍了 DiskLruCache 使用的日志文件 "journal"。日志文件的主要内容是开头的标记行，用来确认日志文件的有效性，以及操作日志。操作日志分为四种：DIRTY、CLEAN、READ 和 REMOVE。每一条日志里，跟在操作标记后面的一串字符是缓存数据的ID，具备唯一性，同时跟本地缓存文件名对应。每一条 CLEAN 日志，在 ID 后面还有几串数字。DiskLruCache 根据初始化参数，给一个缓存数据提供多个缓存文件，CLEAN 日志后面的每一串数字，代表相应缓存文件的大小。

前面提到了两个问题：如何在程序重新启动时恢复缓存数据（包括 LRU 顺序）以及如何避免缓存错误。

READ 日志记录了缓存的 LRU 顺序。DIRTY、CLEAN 和 REMOVE 日志共同保证了缓存的正确性。每一次缓存的读操作都会记录一条相应的 READ 日志。每当要增加一份新的缓存数据时，都会先记录一条相应的 DIRTY 日志，再将数据写入本地文件系统，成功后增加一条 CLEAN 日志，失败时增加 REMOVE 日志。缓存被删除时也会记录一条 REMOVE 日志。DiskLruCache 在初始化时会读取日志文件，删除那些最后操作是 DIRTY 和 REMOVE 的缓存对应的本地文件。

这样，通过一份日志文件，基本上解决了上面提到的两个问题。

下面简单看看 DiskLruCache 的代码。其实大部分的代码都是在处理日志记录。

```java
public final class DiskLruCache implements Closeable {
    
    ......

    private final long maxSize;
    private long size = 0;
    private Writer journalWriter;
    private final LinkedHashMap<String, Entry> lruEntries
            = new LinkedHashMap<String, Entry>(0, 0.75f, true);
    private int redundantOpCount;
    
    ......

}
```

上面这些是 DiskLruCache 的主要数据。maxSize 是初始化是设置的最大缓存空间，size 记录当前已经使用的存储空间，jouralWriter 写日志文件，lruEntries 维护 LRU 顺序，redundantOpCount 记录操作次数。

```java
public final class DiskLruCache implements Closeable {
    
    ......

    /** This cache uses a single background thread to evict entries. */
    private final ExecutorService executorService = new ThreadPoolExecutor(0, 1,
            60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    private final Callable<Void> cleanupCallable = new Callable<Void>() {
        @Override public Void call() throws Exception {
            synchronized (DiskLruCache.this) {
                if (journalWriter == null) {
                    return null; // closed
                }
                trimToSize();
                if (journalRebuildRequired()) {
                    rebuildJournal();
                    redundantOpCount = 0;
                }
            }
            return null;
        }
    };

    ......

    private void trimToSize() throws IOException {
        while (size > maxSize) {
            final Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
            remove(toEvict.getKey());
        }
    }

    ......

    /**
     * We only rebuild the journal when it will halve the size of the journal
     * and eliminate at least 2000 ops.
     */
    private boolean journalRebuildRequired() {
        final int REDUNDANT_OP_COMPACT_THRESHOLD = 2000;
        return redundantOpCount >= REDUNDANT_OP_COMPACT_THRESHOLD
                && redundantOpCount >= lruEntries.size();
    }

    ......

}
```

线程池和 Callable 用来清理缓存文件以及重建日志文件。当 size > maxSize 或 redundantOpCount 到了一定次数后，向线程池里提交一个 Callable。Callable 的 call() 中根据 LRU 顺序删除超过大小的缓存文件，并且重建日志文件。重建时将 lruEntries 的内容写到一个临时文件中, 写完后将临时文件覆盖原日志文件。

这里在 size 超过 maxSize 时，不立即清除缓存，而是提交任务给线程池的原因是，磁盘缓存对空间大小的限制不像内存那么敏感。

```java
public final class DiskLruCache implements Closeable {
    
    ......

    /**
     * A snapshot of the values for an entry.
     */
    public final class Snapshot implements Closeable {
        private final String key;
        private final long sequenceNumber;
        private final InputStream[] ins;

        /**
         * Returns an editor for this snapshot's entry, or null if either the
         * entry has changed since this snapshot was created or if another edit
         * is in progress.
         */
        public Editor edit() throws IOException {
            return DiskLruCache.this.edit(key, sequenceNumber);
        }

        /**
         * Returns the unbuffered stream with the value for {@code index}.
         */
        public InputStream getInputStream(int index) {
            return ins[index];
        }
    }

    ......

    /**
     * Edits the values for an entry.
     */
    public final class Editor {
        private final Entry entry;

        /**
         * Returns an unbuffered input stream to read the last committed value,
         * or null if no value has been committed.
         */
        public InputStream newInputStream(int index) throws IOException {
            synchronized (DiskLruCache.this) {
                ......
                return new FileInputStream(entry.getCleanFile(index));
            }
        }

        /**
         * Returns a new unbuffered output stream to write the value at
         * {@code index}. If the underlying output stream encounters errors
         * when writing to the filesystem, this edit will be aborted when
         * {@link #commit} is called. The returned output stream does not throw
         * IOExceptions.
         */
        public OutputStream newOutputStream(int index) throws IOException {
            synchronized (DiskLruCache.this) {
                if (entry.currentEditor != this) {
                    throw new IllegalStateException();
                }
                return new FaultHidingOutputStream(new FileOutputStream(entry.getDirtyFile(index)));
            }
        }

        /**
         * Commits this edit so it is visible to readers.  This releases the
         * edit lock so another edit may be started on the same key.
         */
        public void commit() throws IOException {
            if (hasErrors) {
                completeEdit(this, false);
                remove(entry.key); // the previous entry is stale
            } else {
                completeEdit(this, true);
            }
        }

        /**
         * Aborts this edit. This releases the edit lock so another edit may be
         * started on the same key.
         */
        public void abort() throws IOException {
            completeEdit(this, false);
        }
    }

    private final class Entry {
        private final String key;

        /** Lengths of this entry's files. */
        private final long[] lengths;

        /** True if this entry has ever been published */
        private boolean readable;

        /** The ongoing edit or null if this entry is not being edited. */
        private Editor currentEditor;

        /** The sequence number of the most recently committed edit to this entry. */
        private long sequenceNumber;

        private Entry(String key) {
            this.key = key;
            this.lengths = new long[valueCount];
        }

        public File getCleanFile(int i) {
            return new File(directory, key + "." + i);
        }

        public File getDirtyFile(int i) {
            return new File(directory, key + "." + i + ".tmp");
        }
    }
}
```

上面是 DiskLruCache 的三个主要内部类，都只截取了部分代码。Snapshot 和 Editor 处理缓存的读写，Entry 记录每一份缓存数据。这几个类的注释也比较详细。

核心的代码就在上面这些以及相关的未截取出来的方法里。
