---
title: Android Handler 原理分析
date: 2016-08-31 10:05:24
tags:
---

## 前言



（该文半年前写于CSDN，回头看看，觉得写的不太好，稍微修改一下） 



平时开发app时，Handler简直已经被用烂了，它的主要工作就是负责子线程何主线程之间的通信。我相信你已经对Handler的使用熟能生巧了，但是你真的了解它吗？

<!-- more -->

## 深入源码



### Handler的使用



先来回想一下我们平时都是怎么使用Handler的。



step1:初始化一个Handler对象，重写其handleMessage()方法：



	Handler mHandler=new Handler() {

        @Override

        public void handleMessage(Message msg){

            //todo

        }

    };

	

step2:发送消息通知Handler处理：



    Message msg=mHandler.obtainMessage();

    msg.what=0;

    mHandler.sendMessage(msg);



我们经常会在主线程中进行step1操作，而在子线程中通过setp2来与主现场通信，如执行UI操作。那么Handler究竟是如何做到线程之间通信的呢？故事要从一个叫Looper的家伙说起。



### Looper是什么



在Android中，对于每一个线程,都可以创建一个Looper对象（最多一个！）和多个Handler。Looper就像一个消息泵，源源不断的从消息池中拿到消息，交给Handler处理。



我们先来简单的看一下Looper类，Looper类中有有四个我们必须要了解的变量：



	private static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

	private static Looper sMainLooper;  // guarded by Looper.class



	final MessageQueue mQueue;

	final Thread mThread;



上面定义了两个静态变量：一个ThreadLocal类型的变量sThreadLocal，一个Looper类型的变量sMainLooper。还有两个final变量：一个队列类型的mQueue，一个线程类型的mThread。这四个变量至关重要。



一个线程想要使用Handler，就必须得创建一个Looper对象。那么创建一个Looper对象需要做什么呢？很简单，一行代码足以。



	Looper.prepare();



在Looper.prepare()中，Looper类会创建一个新的Looper对象,并放入全局的sThreadLocal中。



	sThreadLocal.set(new Looper(quitAllowed));  

	

我们再继续深入看看new Looper(quitAllowed)，很简单，也就两行代码：



	mQueue = new MessageQueue(quitAllowed);

	mThread = Thread.currentThread();

	

原来只是初始化mQueue和mThread这两个变量。	



但是仅仅创建了Looper还不行，还必须开启消息循环，要不然要Looper有何用。开启消息循环同样很简单：



	Looper.Loop();



现在再来看一看Looper.loop()，部分源码已省略：



	 public static void loop() {

        final Looper me = myLooper();

        if (me == null) {

            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");

        }

        final MessageQueue queue = me.mQueue;

        ......

        for (;;) {

            Message msg = queue.next(); // might block

            if (msg == null) {

                // No message indicates that the message queue is quitting.

                return;

            }

            ......

            msg.target.dispatchMessage(msg);

            ......

            msg.recycleUnchecked();

        }

    }



这段代码看起来一堆，其实主要工作也就在于那个for循环。不过还是从第一行代码先看起吧。



第一行调用了函数myLooper(),这个函数的作用在于从sThreadLocal中取出当前线程的Looper对象，因为sThreadLocal为ThreadLocal类型，所以它会保证在多线程情景下，每个线程的数据互不干扰，只能取出自己的Looper对象。



接下来取出Looper对象中的mQueue变量。



	final MessageQueue queue = me.mQueue;



再来看看for循环，大家可以发现这是一个无限循环，没有终止条件。正是这个for循环，开启了我们的消息机制的循环，源源不断的将消息给发送出去：



	Message msg = queue.next();

	msg.target.dispatchMessage(msg);

	

### Handler与Looper的绑定

	

说了这么多，大家对Looper应该稍微有点了解了，但是上述的代码里似乎没有涉及到我们使用的Handler啊，那Looper是如何与Handler进行绑定的呢？又是怎么拿到我们的消息并进行分发的呢？这就要看看Handler的源码了。



先要看一看Handler的构造函数：



	public Handler(Callback callback, boolean async) {

        ......

        mLooper = Looper.myLooper();

        if (mLooper == null) {

            throw new RuntimeException(

                "Can't create handler inside thread that has not called Looper.prepare()");

        }

        mQueue = mLooper.mQueue;

        ......

    }



这里主要有两步操作：首先是获取当前线程的Looper对象，赋值给本地变量，接着将Looper中的消息队列mQueue赋值给Handler的mQueue。通过这两步，Handler就与当前线程的Looper对象绑定了。



再回到Handler的日常使用:



	handler.sendEmptyMessage(int)

	

我们直接深入到这个方法的最底层：sendMessageAtTime(Message msg,long uptimeMills);



	public boolean sendMessageAtTime(Message msg, long uptimeMillis) {

        MessageQueue queue = mQueue;

        if (queue == null) {

            RuntimeException e = new RuntimeException(

                    this + " sendMessageAtTime() called with no mQueue");

            Log.w("Looper", e.getMessage(), e);

            return false;

        }

        return enqueueMessage(queue, msg, uptimeMillis);

    }

    

先是将消息队列mQueue赋值给局部变量queue**（注意注意！这个mQueue指向的可是Looper的mQueue，忘记了请看前述Hander的构造函数）**。再直接看最后一句， **enqueueMessage(queue, msg, uptimeMillis);**这里应该是将我们发送的消息入队了。再向下挖：



	private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {

        msg.target = this;

        if (mAsynchronous) {

            msg.setAsynchronous(true);

        }

        return queue.enqueueMessage(msg, uptimeMillis);

    }

    

果然，这里是将msg放入了Looper的mQueue中了。好了，消息放到队列中去了，有进就有出啊，你个送信的总得把信送出去吧。还记得Loop.loop()方法吗，我们前面说该方法开起来消息机制的循环。loop()的for循环中最终调用了 **msg.target.dispatchMessage(msg)**; 而msg.target, 指向的就是消息的收信人，也就是Handler,那么我们又回到了Handler的源码：



	public void dispatchMessage(Message msg) {

        if (msg.callback != null) {

            handleCallback(msg);

        } else {

            if (mCallback != null) {

                if (mCallback.handleMessage(msg)) {

                    return;

                }

            }

            handleMessage(msg);

        }

    }

    

哈哈，看见了啥？handleMessage(msg) 啊。我们平时new Hander时咋写的记得吧。



	Handler mHandler=new Handler() {

        @Override

        public void handleMessage(Message msg){

            //todo

        }

    };

    

我们重写了handleMessage方法，处理我们接收到的消息，OK，消息终于送达到目的地了！



### 主线程的Looper



最后在说一点，我们平时开发过程中，并没有在Activity中去初始化Looper，那为什么可以使用Handler呢？其实Activity所在的主线程照样也创建了Looper对象，替你干了活而已，虽然它是主线程，也得照样按照规则办事啊。



Activity所在的主线程是ActivityThread，其实这样说并不准确，因为ActivityThread并不是一个线程，它只是主线程的一个入口：ActivityThread中的void main(String[] args)。



	public static void main(String[] args) {

        ......

        Looper.prepareMainLooper();

        ......

        Looper.loop();

        ......

    }



我擦，你看它已经默默做好了一切！

细心的你应该发现了，这里调用的是Looper.prepareMainLooper()，而不是之前所说的Looper.prepare()。是啊，它可是主线程，总得有点不一样的地方吧，岂能完全平起平坐。



我们来挖一挖这个prepareMainLooper()。



	public static void prepareMainLooper() {

        prepare(false);

        synchronized (Looper.class) {

            if (sMainLooper != null) {

                throw new IllegalStateException("The main Looper has already been prepared.");

            }

            sMainLooper = myLooper();

        }

    }

    

它首先也调用了prepare(false)方法，只不过又多做了一件事：**sMainLooper = myLooper()**; 



这时你要问了，这个变量有啥用啊？大家还记得这个变量是静态的吧，这样在子线程中，你就可以通过getMainLooper()来获得主线程的Looper对象了。而对于其他的子线程，因为它们的Looper对象只存储在sThreadLocal中，所以只能够取出自己的Looper对象了。



举个例子，子线程想要在主线程中执行一段代码，就可以按照如下操作：



	new Handler(getMainLooper()).post(new Runnable() {

            @Override

            public void run() {

                //todo

            }

        })



## 总结

以上就是Handler与Looper的哀怨情仇。总得来说，Looper就像是一个邮局，Handler通过sendMessage()将信件交给Looper，放到邮局的仓库mQueue里，邮局Looper再不断的从仓库中取出信，交还给信对应的Handler，收信人调用handleMessage()来读信。