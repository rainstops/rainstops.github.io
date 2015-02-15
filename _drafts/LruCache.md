---
layout: post
title: LruCahe的实现机制与使用
category: "Android"
---
Android应用的开发中,显示图片是几乎是必不可少的一个环节.但是受到内存的限制,这个环节往往会比较痛苦,一个不小心就抛出OutOfMemory异常,导致应用程序崩溃.

Android从API level 12起在android.util包下面提供了[LruCache](http://developer.android.com/reference/android/util/LruCache.html)工具类,帮助我们更好的管理图片占用的内存.在Android Support包中同样能找到这个类.

------

### LruCache的作用
LRU(Least Recently Used),学过操作系统的同学会比较熟悉这个在换页算法中经常被提到的最近最少使用算法.缓存空间已经满的情况下,有新的数据要放进缓存时,需要选取旧的数据移出缓存,LRU算法会将最近没有使用过的数据移出缓存.它基于这样的假设:如果数据最近被访问过,那么将来被访问的概率也更高.

使用LruCache<K, V>缓存图片,初始化时需要设定容量限制.当缓存的图片总大小超过容量限制时,最近没有被使用的图片就会被移出缓存,变成垃圾对象等待回收.这样就不需要手动的管理每张图片所占用的内存.

------

### LruCache的实现机制
整个LruCache类非常简单,只有不到400行代码,其中还有100多行注释.但实际LruCache类中的代码只实现了LRU中管理缓存空间,将旧数据移出缓存的功能.数据存储和LRU顺序是由LruCache类中的一个LinkedHashMap对象实现的.

LinkedHashMap是Java标准类库提供的工具类,继承了HashMap.它同样只有不到400行代码,但实现了LRU顺序.数据存储的功能当然是HashMap类实现的.

那么,LinkedHashMap是如何实现LRU功能的?

其实LRU功能说起来也简单,我们只要把所有的元素放在一个链表中,每使用到一个元素,就把这个元素移动到链表的第一个位置.当有新的元素要加入链表,而链表的容量已满的情况下,将链表最后的元素逐个移除,直到能放入新的元素.而LinkedHashMap就是实现了这样一个链表的HashMap.

这里不得不提到HashMap的数据存储.
HashMap<K, V>的核心数据是一个Entry<K, V>数组. Entry<K, V>是HashMap的一个内部类,下面是它的数据定义:
{% highlight java %}
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        final int hash;
{% endhighlight %}
Entry<K, V>主要是存放了Key, Value和一个next引用. Key, Value的作用不需解释, next引用是为了解决哈希碰撞.
HashMap对每一个新Key的put(key, value)操作,都会生成对应的Entry<key, value>,根据Entry的hash值(即Key的哈希值)将这个Entry放入Entry数组中对应的位置.如果Entry数组的这个位置已经有一个Entry了,就发生了哈希碰撞.此时将新Entry的next引用指向旧Entry,然后将新Entry放到数组的该位置.这样,发生哈希碰撞的Entry组成了一个链表.这种哈希碰撞的解决方案又叫做开散列.

![HashMap](http://7vzocb.com1.z0.glb.clouddn.com/image/blog/HashMap_structure.png)

每一种Collection类都实现了Iterator接口, HashMap也有自己的迭代顺序.
HashMap的迭代会从数组头开始,找到第一个非空位置,访问这个位置的Entry,紧接着,通过Entry的next引用访问该位置上所有碰撞的Entry.然后再继续访问数组后面的Entry.
