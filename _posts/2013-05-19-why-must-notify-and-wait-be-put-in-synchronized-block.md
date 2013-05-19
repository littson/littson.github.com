---
layout: post
title: "为什么wait()/notify()必须被放在synchronized块中?"
description: ""
category: 
tags: [java, monitor, synchronized, wait, notify]
---
{% include JB/setup %}

`java.lang.Object`提供了`wait()`/`notify()`方法,用于支持线程之间的协同. 例如在生产者-消费者模型中, 当生产者线程生产数据后, 需要通知消费者线程进行消费(`notify()`). 当消费者线程消费完已有数据时, 需要等待(`wait()`). 一个典型的示例代码如下:

	class Producer implements Runnable {

        List<Object> data;
        
        @Override
        public void run() {
            while (true) {
                synchronized (data) {
               	 	// produce data
                	// data.add("data" + System.currentTimeMillis());
                	data.notify();
                }
            }
        }

    }
	
	class Consumer implements Runnable {

        List<Object> data;

        @Override
        public void run() {
            while (true) {
                synchronized (data) {
                    if (data.isEmpty()) {
                        try {
                            data.wait();
                        } catch (InterruptedException e) {
                        	continue;
                        }
                    }
                    // consume data
                    // ...
				}
            }
        }

    }

我们注意到,消费者线程需要做两步操作:

1. 检查是否有数据(`data.isEmpty()`).
2. 如果没有数据,等待(`data.wait()`).

这两个操作不是原子的,如果我们不使用`synchronized`,生产者线程有可能在1和2之间插入执行:

1. 准备数据(`data.add()`).
2. 通知消费者(`data.notify()`).

因为此时消费者线程还没进入等待状态,这个通知相当于丢失了.导致有数据待消费,但消费者线程在等待数据的场面.

通过以上分析可以知道,我们其实只是需要一个排他锁. 那么,我们直接使用一个锁来替代`synchronized`,比如使用`java.util.concurrent.locks.Lock`是否可以呢(请忽略它出现的时间)? 

答案是不可以. 

原因是,当消费者线程在调用`wait()`前,它已经持有了排他锁. 此时生产者线程无法获取到锁,也无法调用`notify()`. 

为了解决这个问题,我们在调用`wait()`时需要原子的完成两个操作:

1. 释放锁.
2. 让当前线程进入等待状态.

类似的,在调用`notify()`时需要:

1. 释放锁.
2. 把等待状态的一个线程恢复为可运行状态.

也就是说,`wait()`/`notify()`隐含了锁的释放操作,而`synchronized`隐含了锁的获取操作,这解释了它们为什么必须出现在一块.

P.S. `synchronized`/`wait()`/`notify()`来源于[monitor](http://en.wikipedia.org/wiki/Monitor_(synchronization\)). 在Java中,每个对象都有一个关联的`monitor`,使用`synchronized`即使用了`monitor`的排他性,使用`wait()`/`notify()`即使用了`monitor`的`condition variable`. 在`java.util.concurrent`中,JDK终于独立提供了`monitor`:[Lock](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/locks/Lock.html)和[Condition](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/locks/Condition.html).

---
EOF
