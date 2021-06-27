# sentinel

  哨兵是Redis的高可用性的解决方案,由一个或多个Sentinel实例组成的Sentinel系统可以监视伤科昂然个主服务器,以及这些主服务器的所有从服务器,并在
被监视的主服务器进入下线状态时,自动将下线主服务器下的某个从服务器升级为新的主服务器,然后由新的主服务器代替已下线的主服务器继续处理命令请求,

![](https://res.weread.qq.com/wrepub/epub_622000_249)

![](https://res.weread.qq.com/wrepub/epub_622000_250)

当server1的下线时长超过用户设定的下线时长上限时,Sentinel系统就会对server1执行故障转移操作
  1. 首先Sentinel系统会挑选Server1下的其中一个从服务器,并将这个被选中的从服务器升级为新的主服务器.
  2. 之后Sentinel会向server1下的所有从服务器发送新的复制指令,让他们成为新的主服务器的从服务器,当所有的从服务器都的主服务器时,故障转移操作执行完毕
  3. Sentinel还会继续监视已下线的server1,关在它重新上线时,将它设置为新的主服务器的从服务器.

![故障转移](https://res.weread.qq.com/wrepub/epub_622000_251)
![当server1重新上线后会复制server2](https://res.weread.qq.com/wrepub/epub_622000_252)
