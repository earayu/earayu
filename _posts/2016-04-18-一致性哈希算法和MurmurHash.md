---
title: 一致性哈希算法和MurmurHash
updated: 2016-04-18 21:22
---


### 一致性哈希算法

* 为什么要发明*一致性哈希算法？*

  * 拿分布式系统举例：文件存储在不同的N台服务器上，我们怎么快速知道文件在哪个服务器上呢？可以先计算得出文件的hash值，记为`hashValue`，然后用hashValue对N取模，得到的号码就是文件所在的服务器的号码。但是这样会出现一个问题：增加或减少服务器数量的时候，`hashValue/(N-1)`会和`hashValue/N`有很大不同，`hashValue`也就不能再定位到服务器了。而一致性哈希算法的出现，解决了这个问题。

    关于一致性哈希算法的原理，可以看这两篇文章：

    * [聊聊一致性哈希](https://zhuanlan.zhihu.com/p/24440059)   

    * [一致性哈希算法与Java实现](http://www.blogjava.net/hello-yun/archive/2012/10/10/389289.html)

      ​

### MurmurHash

>MurmurHash算法不能用于加密，可以用作hash-based查找方法。Austin Appleby在2008年发明了MurmurHash，后又推出了Murmurhash2和Murmurhash3。使用它的开源项目有：nginx, Hadoop, Redis, npm, Guava, libstdc++, Memcached 等等。



* Murmurhash1
  * 被废弃


* Murmurhash2
  * MurmurHash2A
  * MurmurHash64A
    * 为64位处理器优化
  * MurmurHash64B
    * 为32位处理器优化
  * MurmurHash2-160
    * 生成160位hash值


* Murmurhash3

  当前版本，可以生成32位和128位的hash值。使用128位版本的时候，因为在各自平台做了优化，所以x86和x64机器不提供一样的hash值。

#### 性能比较

|      算法       | 效率(mb/sec)  |
| :-----------: | :---------: |
|  OneAtATime   | 354.163715  |
|      FNV      | 443.668038  |
| SuperFastHash | 985.335173  |
|    lookup3    | 988.080652  |
|  MurmurHash1  | 1363.293480 |
|  MurmurHash2  | 2056.885653 |
|  MurmurHash3  |     没找到     |