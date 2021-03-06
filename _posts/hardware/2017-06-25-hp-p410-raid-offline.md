---
layout: post
title:  "HP阵列offiline的处理"
date:   2017-06-25 15:05:00 +0800
categories: "硬件"
tag: ["HP服务器"]
---

* content
{:toc}


原文转载自[地址](http://blog.chinaunix.net/uid-25800-id-3368203.html)


内容如下

    给大家说下一个HP的阵列卡 还有一些低端存储（例如MSA1000）阵列数据恢复经验   

    可能不经意的对阵列进行一些操作时（例如更换硬盘）导致阵列损坏    标
    志是该阵列在相关工具（例如HP的ACU）查看下状态为Failed，这时该阵列中的硬盘数据已被锁定，
    处于不可读写的状态，大多数状态下数据都还是完好的，
    也就是说你把阵列上的硬盘恢复到你操作前状态（一定要记住位置不能错）是可以恢复数据的，不用再去数据恢复中心，可以省一笔银子。
    解锁的方法就是使用HP的阵列管理工具ACU，点击状态为Failed的逻辑卷（就是打X的逻辑卷），
    这时会多出一个选项是Re-enable Failed logical Drive，点击之后出一个对话框是警告你之类的，
    不要管，直接确定就OK，然后再看下你坏的逻辑卷，是不是数据回来了，
    该方法我们多次测试过，全部成功，但是你拨打HP客服电话的时候他会告诉你是有几率的，因为HP不会去担这个责任。
    最后说一句，对存储的一切操作都是有风险，大家要小心，上面是我的个人经验，希望对大家有用，备份数据很重要。


以下是个人说明

    在raid5的情况下
    HP整列卡检测到有硬盘处于将要损坏的状态(硬盘smart数据表面坏块过多之列)
    突然挂掉了另外一个盘(raid5 不能坏2个盘)
    阵列卡会自动offline以确保整个阵列数据安全

    虽然这种情况下数据还是正常的,但是系统提示和报告中都会表示阵列所有数据丢失

    这时候我们只要在上面ACU里强制上线raid即可,当然,处于即将损坏状态的盘不能拔掉
    先替换掉已经挂掉的盘,阵列数据恢复好以后再替换即将损坏的盘

    必须下载HP的smart start,通过里面的ACU才能强制上线
    只靠服务器里的阵列卡控制器是无法强制上线的
