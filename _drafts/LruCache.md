---
layout: post
title: LruCahe的实现机制与使用
category: "Android"
---
Android应用的开发中,显示图片是几乎是必不可少的一个环节.但是受到内存的限制,这环节往往会比较痛苦,一个不小心就抛出OutOfMemory异常,导致应用程序崩溃.

Android从API level 12起在android.util包下面提供了[LruCache](http://developer.android.com/reference/android/util/LruCache.html)工具类,帮助我们更好的管理图片占用的内存.在Android Support包中同样能找到这个类.

### LruCache的作用
LRU(Least Recently Used)学过操作系统的同学会比较熟悉这个在换页算法中经常被提到的最近最少使用算法.缓存空间已经满的情况下,有新的数据要放进缓存时,需要选取旧的数据移出缓存,LRU算法会将最近没有使用过的数据移出缓存.它基于这样的假设:如果数据最近被访问过,那么将来被访问的概率也更高.

使用LruCache<K, V>缓存图片,初始化时需要设定容量限制.当缓存的图片总大小超过容量限制时,最近没有被使用的图片就会被移出缓存,变成垃圾对象等待回收.这样就不需要手动的管理每张图片的大小.

### LruCache的实现机制
整个LruCache类非常简单,只有不到400行代码,其中还有100多行注释.LruCache类的核心数据是一个LinkedHashMap对象,LinkedHashMap类实现了数据存储和LRU功能.

LinkedHashMap是Java标准类库提供的工具类,继承了HashMap.它同样只有不到400行代码,但是实现了LRU功能.数据存储的功能当然是HashMap类实现的.
