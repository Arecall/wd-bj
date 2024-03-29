---
title: 近几年Linux TCP相关的漏洞被夸大了
date: 2019-07-08 22:45:19
tags: CSDN迁移
---
  本文不提CVE-2019-11477 SACK Panic漏洞，没意思。本文以CVE-2019-11478做引。

 当我们最近在为CVE-2019-11478这个漏洞而心惊胆战的时候，其实我们早就已经忘了不久之前的另一个几乎同样的漏洞：

  
  * CVE-2019-11478：数据发送端遍历排序SACK而被DDoS。 
  * CVE-2018-5390：数据接收端遍历排序OFO而被DDoS。  简直是孪生兄弟。是碰巧吗？不，这非常可以预期。

 根源就是TCP会乱序：

  
  * 接收端乱序接收的数据存于Out of order队列，即OFO队列。 
  * 接收端将OFO的接收情况选择最多4段以SACK的形式告诉数据发送端。  由于乱序发生在端到端TCP不知情的中间网络，这就意味着：

  
  * OFO数据端可以伪造。 
  * SACK段可以伪造。  紧接着看：

  
  * 伪造的OFO数据端可以促使数据接收端去处理它。 
  * 伪造的SACK段可以让数据的发送端去处理它。  好了，到此为止，看样子两端都需要处理这些伪造的报文了。处理就处理呗，有问题吗？

 有问题，问题就在于TCP是一个串行字节流，为了实现的方便，用链表实现那是最直接最应景的：

  
  * 接收端的OFO队列使用链表实现。 
  * 发送端的重传队列使用链表实现。  现在应该可以看出来有什么问题了吧。如果可以伪造一些OFO段和SACK段，让这些链表遍历更狠些，岂不是可以多消耗点无论接收端还是发送端的CPU资源吗？这正是利用了链表遍历的O(n)O(n)O(n)低效率特征。【真的就是因为链表遍历和排序，要是再不信，非要觉得另有蹊跷，我也没办法了】

 就这么简单。

 攻击数据发送端的CVE-2019-11478我已经做出demo，但却很难实施，我当时说过：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190708214906866.png)  
 你想用一个脚本AB测试试试看CPU是不是高了，确实，高了一丢丢，但是想要促成真正的DDoS，难点根本就不在TCP，而在第一个D。想DDoS攻击，使用成熟的死亡之ping(ping of death, POD)岂不是更加简单吗？！搞什么TCP。

 写这篇文章的缘由其实是因为今天在路上看到了另一篇评价这个事的文章，读后感来着。这篇文章是 _**《Linux 内核 TCP 漏洞被夸大了，两周前已经修复》**_ ，它的链接：  
 [https://www.oschina.net/news/98831/linux-kernel-tcp-bug-fixed](https://www.oschina.net/news/98831/linux-kernel-tcp-bug-fixed)

 我主要是看到了最后一段才有感而发写了此文：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190708215349376.png)  
 程序员可能根本就不接受什么拔网线，拔电源线这种 _**low逼**_ 的做法，只有写程序把服务器搞瘫才是 _**正道**_ 。这是职业本性，无可厚非，但是说到危害，到底哪个更具有威胁性，最具有破坏性价比呢？

 说实话，我怕有人怼我说我奇技淫巧所以没敢说拔电源线之类的策略，所以我推荐死亡之ping，但事实上，拔线可能真的会更需要技巧。

 你如何进入机房？靠轻功，穿墙术，还是说靠贿赂管理员，或者阴谋阳谋？随便哪一点都不是一个普通人可以胜任的，我这里说的普通人包括各种职业的，程序员，经理，工人，打手，富豪，要饭的之类。

 这些留给特工吧，我们凡人说回技术本身。

 再看上述我引用文章里的约束，你如何维持大量的全双工TCP连接，这里有另一个漏洞要比这两个DDoS漏洞要有意义的多：  
 [http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-5696](http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-5696)  
 这里有篇博客：  
 [https://bobcares.com/blog/fix-linux-off-path-tcp-attacks-cve-2016-5696/](https://bobcares.com/blog/fix-linux-off-path-tcp-attacks-cve-2016-5696/)  
 这个就是教你如何劫持一个连接的，也并不难理解，相反，它很简单。即便如此，你能利用吗？

 如果实在是懒得看RFC5961，那就大摆拳王八拳来一通，只要足够密集，总是会有中招的，打TCP，30多年前的协议，就是要用徐晓冬的技巧，一顿密集大摆拳怼上去就行，没啥技巧。

 诱导什么诱导，只要密集，仅凭猜都能八九不离十，序列号空间也就4G，而且现如今的带宽越来越大，主机内存越来越大，这意味着拥塞窗口，接收缓冲区都会越来越大，那么in window将越来越容易命中。

 架一杆机关枪乱打，当目标越来越多越来越密的时候，你的命中率会越来越高，虽然，你还是你，什么都没做，也什么都不用做。哈哈。

 
--------
 说真的，想把服务器搞宕机，还真的挺难，但是无非根本目的就是想让它不能用呗，那么把它的CPU搞高便是了，这是一道比把机器搞宕机更简单的题目。

 可是即便如此，为什么都老是盯着TCP？

 我有好多种方法把机器的CPU搞高，为什么这些没有被任何一个人曝出CVE，为什么曝出CVE的总是那些要么根本就没法利用，要么利用条件很苛刻几乎满足不了的TCP相关的？

 使用CentOS 7的不在少数吧，换句话说，使用4.3内核版本之前内核的不在少数吗？你们知道系统的定时炸弹吗？David Miller或者在他review通过情况下引入了不下三个可以报CVE的代码段，这个熟人社区有人吭气吗？

 姑且这些和David Miller有关的缺陷就认为是三个吧，没有一个和TCP有关，那如果说和TCP有关的，那就算第四个吧，恰好，真是蹭热点，CVE-2019-11478的修补patch和David Miller有关，然而，有意思的是这个patch竟然是错的，耍了圈里一票人啊！非常混乱。

 如何把CPU搞高，详见：  
 [https://blog.csdn.net/dog250/article/details/91046131](https://blog.csdn.net/dog250/article/details/91046131)  
 [https://blog.csdn.net/dog250/article/details/91047124](https://blog.csdn.net/dog250/article/details/91047124)  
 第三个恕不能提供。

 
--------
 回到主题，TCP的漏洞真的被夸大了，夸大的原因可能仅仅是因为TCP受关注，夸大的原因还有一个就是很多人真的不懂。一群人觉得某某成了时髦，当然会天天吹捧，其实自己也是什么都不懂，只是关注大多数人关注的东西罢了。

 不过为什么没人去不懂装懂扯扯ARP，802.3ad PDU，GBP这些呢？因为这些相关的从业人员太小众，反过来看看TCP的受众，不提非IT互联网行业的终端用户，就看看BAT这些大厂的职员有多少，哪个不天天和TCP打交道。懂或者不懂不重要了，重要的是，TCP火了。

 谢特的，我写这篇文章，依然还是用了TCPCPTCP！

 出血丝血丝出血丝！(这是我小学时发明的玩法，被抓到蛋时，仔细观察被抓者瞪大的眼睛，里面布满了血丝)

 
--------
 浙江温州皮鞋湿，下雨进水不会胖。

   
  