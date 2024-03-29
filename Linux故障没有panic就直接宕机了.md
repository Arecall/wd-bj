---
title: Linux故障没有panic就直接宕机了
date: 2019-07-11 23:00:21
tags: CSDN迁移
---
  一般情况下，Linux内核在遇到不可恢复的故障时，就会panic，类似Windows的蓝屏，不管是panic还是蓝屏，总是会在屏幕上打印一些东西，这就是 _**最后一屏有用的信息。**_

 这最后一屏信息非常重要，它可以帮助我们定位引发系统故障的潜在的bug。事实上，很多时候，对于开源系统而言，这最后一屏信息甚至比vmcore之类的更加有用且轻量级。比如根据堆栈和出错的函数位置，基本对着代码就能看出个究竟。

 那么，问题是，是不是Linux系统遇到不可恢复的故障，总是会panic呢？有没有什么情况下是直接就宕机而没有任何遗言呢？或者说故障太严重，以至于根本就没有任何机会留下任何信息就宕机了。

 肯定是有的。

 比如电源故障，掉电之类，计算机赖以生存的能源都没有了。还有就是主板上泼水之类，直接破坏硬件组织。不过这些都是系统外部的严重事件，我指的是，只给你个键盘，你能否写出一个瞬间宕机的程序呢？

 当然了，前提条件必须是故障触发，而不能是调用底层的reboot，halt等接口。换句话说，你能否通过 _**踩内存**_ 的方式让机器绕过panic直接宕机或者重启呢？

 肯定可以。如何来做呢？让我们想一下panic函数赖以执行的条件是什么。

 我们知道，在现代操作系统中，即便是内核里的panic函数也是用虚拟地址取指载入CPU来执行的，也就是说，panic函数的指令是通过虚拟地址来寻址的，在现代操作系统中，虚拟地址和物理地址之间的纽带就是MMU，虚拟地址通过页表翻译成物理地址发射到地址总线去内存中取数据，这一切工作的前提，全凭一个指向当前进程页表的CR3寄存器。

 panic函数的指令也是靠CR3寄存器指示的页目录逐渐被找到然后载入CPU去执行的，那么很简单，只需要破坏掉CR3寄存器即可。即切断虚拟地址和物理内存的纽带！

 写个模块，init函数里包含下面的函数：

 
```
unsigned long var = 0;
asm volatile ("mov %0, %%cr3"::"r"(var));

```
 系统秒崩。你试试。

 不过，你直接操作寄存器，不能算数，那就换一种方法。我们知道在进程切换的时候，会执行load CR3的动作，类似上面的汇编代码，将新的进程的页目录物理地址载入CR3寄存器，那么给它改了就行了啊：

 
```
list_for_each(pos, &ts->tasks) {
	p = list_entry(pos, struct task_struct,tasks);
	if (p->pid == 1339) { // 随便一个非当前insmod进程即可
		p->mm->pgd = PAGE_OFFSET + 123456;
	}
}

```
 当进程切换到1339号时，系统秒崩。

 
--------
 我们换个角度，如果在内核空间发生了踩内存，恰恰把某个task的pgd给改成了一个非页目录的地址，会发生什么呢？当然是秒崩咯。

 所以这类bug非常难以定位，因为你连最后一屏信息都拿不到，更别说什么vmcore了，虚拟地址和物理内存之间的纽带就像大动脉一样被砍掉，系统没有任何机会去留下哪怕一点点信息，因为留下哪怕一点点信息这件事也是指令，也需要虚拟地址去寻址的。

 可见，想要搞点破坏，需要对操作系统以及平台系统结构有着深入的理解，不然，在不砸机器不浇水不把电源的约束下，只给个键盘是无法对系统造成破坏的。

 这里，留下一个思考：  
 _**如何写一个程序(内核的或者用户态的)，清空系统所有的操作系统可见的物理内存呢？**_  
 这个问题并不容易！

 
--------
 突然就又想到了panic和oops字面的意思。

 panic是惊恐不知所措的意思，既然不知所措了，那就待在原地最好最安全，所以panic的最后是一个无限的循环，然后里面就是闪灯之类的操作。

 oops是 _**哎哟**_ 的意思，就好像不知怎么回事有人背后撞了你一下，你毫无防备和预期，然后感觉到了一点疼，哎哟一声。这个oops显然没有panic严重。

 可是有些时候，oops的同时也伴随着panic，这也可以理解。背后被撞了一下，那就哎哟一声罢了，这就是oops，如果背后被捅了一刀，或者后心窝被打了一枪，那肯定就是先哎哟一声，然后惊恐倒地，身体发生了预期之外不可恢复的故障，就是oops和panic了。

 再举一个例子，前些天Robin Li老板被浇水，那就只是oops，而换成几十年前的肯尼迪，那就是oops/panic了。

 那么，什么是load 0 to CR3呢？斩首吗？貌似也不，因为研究表明身首分离后，人还是有意识的，那么到底是什么能让人一瞬间就结束呢？目前貌似还没有还不知道。

 
--------
 浙江温州皮鞋湿，下雨进水不会胖。

   
  