---
title: Android Binder解密
date: 2016-09-10 23:24:50
tags:
---

## 前言
稍微看过Android FrameWork层的人应该都知道Binder，因为app与系统服务之间的通信基本上都是建立在Binder的基础上。之前对Binder也是云里雾里，似懂非懂，于是花了不少时间，看了很多资料和源码，才大致了解了Binder通信的原理，总结出来，如有错误，还望指正。

<!-- more -->

## 简介
Binder是什么？Binder是为跨进程通信而生的产物。众所周知，我们的app都是由Zygote进程fork出来的，每个app都运行在单独的进程中，一个app想与另外一个app进行通信，只能采用跨进程的方式，传统的Linux跨进程方式有如下几种：管道、信号量、共享内存及Socket等，Android系统建立在Linux的基础上，但其采用的是Binder来进行IPC的。

接下来让我们用一个小Demo展示一下如何用Binder进行跨进程通信。

## 实例
既然是跨进程通信，那么至少得有两个进程，最简单的方式就是指定activity的process属性，但是本文为了更清楚的讲解binder，采用的方式是建立两个app，一个作为服务端，一个作为客户端。服务端提供一个简单的生词本功能，客户端可以向服务端插入和查询生词。
使用Binder通信主要有以下几个步骤：
### Step1:编写AIDL文件
想要使用Binder，必须要先了解AIDL(Android Interface Definition Language)，也就是接口定义语言，提供接口给远程调用者。
为了给客户端提供生词本的调用接口，我们在/src/main目录下先新建一个文件夹aidl,并新建一个aidl文件IDictionaryManager.aidl。

	// IDictionaryManager.aidl
	package com.wanginbeijing.dictionaryserver;

	interface IDictionaryManager {
      	void add(String chinese,String english);
    	String query(String chinese);
	}
接口中提供了两个方法：add()和query()，分别作为插入和查询操作。Build一下工程，android studio会自动为我们生成一个java类：IDictionaryManager.java。
![](http://od35ecbnl.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-06%20%E4%B8%8B%E5%8D%8810.03.15.png)

我们来看看这个java类里面都写了什么
	
	public interface IDictionaryManager extends android.os.IInterface {
	
    	public static abstract class Stub extends android.os.Binder implements com.wanginbeijing.dictionaryserver.IDictionaryManager {
        	private static final java.lang.String DESCRIPTOR = "com.wanginbeijing.dictionaryserver.IDictionaryManager";

        	public Stub() {
            	this.attachInterface(this, DESCRIPTOR);
        	}


        	public static com.wanginbeijing.dictionaryserver.IDictionaryManager asInterface(android.os.IBinder obj) {
            	if ((obj == null)) {
                	return null;
            	}
            	android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            	if (((iin != null) && (iin instanceof com.wanginbeijing.dictionaryserver.IDictionaryManager))) {
                	return ((com.wanginbeijing.dictionaryserver.IDictionaryManager) iin);
            	}
            	return new com.wanginbeijing.dictionaryserver.IDictionaryManager.Stub.Proxy(obj);
        	}

        	@Override
        	public android.os.IBinder asBinder() {
            	return this;
        	}

        	@Override
        	public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            	switch (code) {
                	case INTERFACE_TRANSACTION: {
                    	reply.writeString(DESCRIPTOR);
                    	return true;
                	}
                	case TRANSACTION_add: {
                    	data.enforceInterface(DESCRIPTOR);
                    	java.lang.String _arg0;
                    	_arg0 = data.readString();
                    	java.lang.String _arg1;
                    	_arg1 = data.readString();
                    	this.add(_arg0, _arg1);
                    	reply.writeNoException();
                    	return true;
                	}
                case TRANSACTION_query: {
                    	data.enforceInterface(DESCRIPTOR);
                    	java.lang.String _arg0;
                    	_arg0 = data.readString();
                    	java.lang.String _result = this.query(_arg0);
                    	reply.writeNoException();
                    	reply.writeString(_result);
                    	return true;
                	}
            	}
            	return super.onTransact(code, data, reply, flags);
        	}

        	private static class Proxy implements com.wanginbeijing.dictionaryserver.IDictionaryManager {
            	private android.os.IBinder mRemote;

            	Proxy(android.os.IBinder remote) {
                	mRemote = remote;
            	}

            	@Override
            	public android.os.IBinder asBinder() {
                	return mRemote;
            	}

            	public java.lang.String getInterfaceDescriptor() {
                	return DESCRIPTOR;
            	}

            	@Override
            	public void add(java.lang.String chinese, java.lang.String english) throws android.os.RemoteException {
                	android.os.Parcel _data = android.os.Parcel.obtain();
                	android.os.Parcel _reply = android.os.Parcel.obtain();
                	try {
                    	_data.writeInterfaceToken(DESCRIPTOR);
                    	_data.writeString(chinese);
                    	_data.writeString(english);
                    	mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                    	_reply.readException();
                	} finally {
                    	_reply.recycle();
                    	_data.recycle();
                	}
            	}

            	@Override
            	public java.lang.String query(java.lang.String chinese) throws android.os.RemoteException {
                	android.os.Parcel _data = android.os.Parcel.obtain();
                	android.os.Parcel _reply = android.os.Parcel.obtain();
                	java.lang.String _result;
                	try {
                   	 _data.writeInterfaceToken(DESCRIPTOR);
                    	_data.writeString(chinese);
                    	mRemote.transact(Stub.TRANSACTION_query, _data, _reply, 0);
                    	_reply.readException();
                    	_result = _reply.readString();
                	} finally {
                    	_reply.recycle();
                    	_data.recycle();
                	}
                	return _result;
            	}
        	}

        	static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        	static final int TRANSACTION_query = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    	}

    	public void add(java.lang.String chinese, java.lang.String english) throws android.os.RemoteException;

    	public java.lang.String query(java.lang.String chinese) throws android.os.RemoteException;
	}

这个接口类比较长，继承了android.os.IInterface这个接口。这个类简化的结构大致如下：

	public interface IDictionaryManager extends android.os.IInterface {
	
    	public static abstract class Stub extends android.os.Binder implements com.wanginbeijing.dictionaryserver.IDictionaryManager {
        
        	public Stub() {
            	this.attachInterface(this, DESCRIPTOR);
        	}


        	public static com.wanginbeijing.dictionaryserver.IDictionaryManager asInterface(android.os.IBinder obj) 
        	
        	@Override
        	public android.os.IBinder asBinder() 
        	
        	@Override
        	public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
        	
        	private static class Proxy implements com.wanginbeijing.dictionaryserver.IDictionaryManager {
            	private android.os.IBinder mRemote;

            	Proxy(android.os.IBinder remote)
            	
            	@Override
            	public android.os.IBinder asBinder() 
            	
            	public java.lang.String getInterfaceDescriptor() 
            	
            	@Override
            	public void add(java.lang.String chinese, java.lang.String english)
            	
            	@Override
            	public java.lang.String query(java.lang.String chinese)  
          }

        	static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        	static final int TRANSACTION_query = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    	}

    	public void add(java.lang.String chinese, java.lang.String english) 

    	public java.lang.String query(java.lang.String chinese)
	}

该类首先包含了一个抽象内部类：Stub, 该类继承自Binder并实现了IDictionary接口。在Stub的内部，又包含了一个静态内部类：Proxy，Proxy类同样实现了IDictionary接口。

### Step2:定义一个Service，用于客户端连接

	public class DictionaryManagerService extends Service {
    	private Map<String, String> mMap = new HashMap<>();

    	@Nullable
    	@Override
    	public IBinder onBind(Intent intent) {
        	return new IDictionaryManager.Stub() {
            	@Override
            	public void add(String chinese, String english) throws RemoteException {
                	mMap.put(chinese, english);
                	Log.e("DictionaryManager", "add new word");
            	}

            	@Override
            	public String query(String chinese) throws RemoteException {
                	return mMap.get(chinese);
            	}
        	};
    	}
	}

该类中定义了一个HashMap用来保存生词，并重写了Service中的onBind方法，在onBinder方法中，返回了一个继承自IDictionaryManager.Stub的匿名内部类，并重写了IDictionaryManager接口中add和query方法，实现真正的生词本业务逻辑。
### Step3:客户端连接服务端Service
客户端要调用服务端接口，先要将服务端定义好的aidl文件拷贝到客户端相同目录下，并build生成java文件。然后开始连接服务器端：

	public class MainActivity extends Activity {

    	private IDictionaryManager mDictionaryManager;

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_main);
        	Intent intent = new Intent();
        	intent.setAction("android.intent.action.DictionaryManagerService");
        	intent.setPackage("com.wanginbeijing.dictionaryserver");
        	bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
        	//添加一个新单词
        	findViewById(R.id.btn_add).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View view) {
                	try {
                    	mDictionaryManager.add("你好", "Hello");
                	} catch (RemoteException e) {
                    	e.printStackTrace();
                	}
            	}
        	});

        	//查询单词
        	findViewById(R.id.btn_query).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View view) {
                	try {
                    	String english = mDictionaryManager.query("你好");
                    	Toast.makeText(MainActivity.this, english, Toast.LENGTH_SHORT).show();
                	} catch (RemoteException e) {
                    	e.printStackTrace();
                	}
            	}
        	});
    	}

    	private ServiceConnection mConnection = new ServiceConnection() {
        	@Override
        	public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            	IDictionaryManager dictionaryManager = IDictionaryManager.Stub.asInterface(iBinder);
            	try {
                	mDictionaryManager = dictionaryManager;
                	Toast.makeText(MainActivity.this, "connect success", Toast.LENGTH_SHORT).show();
            	} catch (Exception e) {
                	Toast.makeText(MainActivity.this, "connect failed", Toast.LENGTH_SHORT).show();
                	e.printStackTrace();
            	}
        	}

        	@Override
        	public void onServiceDisconnected(ComponentName name) {
        	}
    	};
	}
	
客户端的MainActivty的onCreate()方法中，首先自动连接远程service，拿到远程service传回的Binder对象后，强转为IDictionaryService类型的变量mDictionaryManager。然后在add按钮和query按钮的监听事件中分别添加和查询单词（你好，Hello）。下面展示一下操作Demo.
![](http://od35ecbnl.bkt.clouddn.com/demo.gif)
可以看见，在连接远端service成功后，首先点击add按钮插入单词，接着调用query接口，可以成功查询到刚刚插入的单词。

以上就是使用Binder进行IPC的主要过程，但是仅仅掌握了使用方法怎么能满足我们的好奇心呢，我们还必须要挖挖它的原理。

## 原理
### java层
说到原理，还得先从代码看起，回到上述所说的Android Studio自动生成的那个java类:IDictionaryManager。刚才已经说过，该类中有一个继承自Binder的Stub类，它实现了我们所需要的IDictionaryManager接口。我们先来看看它的几个方法：

	public static IDictionaryManager asInterface(IBinder obj)
这方法将一个IBinder对象转换成实现了IDictionaryManager。如果请求调用的客户端在同一进程，就直接返回Stub对象本身，如果不在同一进程，就返回Stub类中的Proxy对象。Proxy类同样实现了IDictionaryManager接口。

	asBinder
这个方法很简单，就直接返回了Stub对象自己。

	onTransact
这个方法主要是进行用来调用其他函数的，它会根据传入的参数code，选择调用add方法还是query方法。

接下来看看Stub.Proxy类中的方法

	asBinder
	
这个方法返回了Proxy对象中的mRemote变量，mRemote变量的赋值发生在Proxy类的构造函数中，而在Stub类的asInterface()方法中调用了该构造函数Proxy()，并传入了一个Binder对象，将mRemote指向了这个Binder对象。

	add
该add方法继承自IDictionary接口。里面主要是将客户端传参写入_data中,然后调用transact()方法，在transact方法中，会回调Stub类中的onTransact()。

	query
同上述add方法一样，只不过多了一个返回值。

介绍完这些方法之后，我们再来将这些方法串起来，讲一下完整的IPC过程。客户端连接远程Service时，Service会返回一个Binder对象，即我们在DictionaryManagerService.onBind()中返回的继承自Stub的匿名对象，该类实现了生词本逻辑。
客户端调用有两种情况：

**1:客户端和服务端在同一个进程**

服务器端就直接将该Stub对象返回，即我们在onBind()中返回的Stub对象，客户端调用返回对象的asInterface()方法，将Stub对象强转为IDictionManager对象，直接调用IDictionManager对象中的方法，就像调用本地对象的方法一样。

**2:客户端和服务端不在同一个进程**

因为是跨进程，两端的内存资源是不能共享的，服务器端不可能将真正的Stub对象返回，它只能将Stub中的代理类Prxoy对象返回。客户端同样调用返回对象的asInterface()方法，将Proxy对象强转为IDictionManager对象，所以这次调用IDictionManager中add方法的时候，调用的实际上是Prxoy.add()。这次调用发生在客户端进程，然后在Proxy.add()方法中，将参数和目标函数code写入Binder驱动中，并通过回调服务端Stub.onTransact()方法，实现调用真正的Stub.add(）方法。

### C++层

我们开发时所见到的Binder是Android系统提供给我们的java接口，java层的Binder对象只是Android对底层Binder的一个封装，提供给上层开发人员使用，真正的Binder其实隐藏在系统底层，默默的替我们进行着跨进程通信。

Java层的服务端Binder对象在C++层对应的对象为BBinder，而客户端拿到的BinderProxy对象对应的则为BpBinder，BBindder和BpBinder都继承自IBinder，上层Binder和BinderProxy之间的通信其实是BBinder和BinderProxy之间的通信。

**Binder的核心是Binder Driver**，即Binder驱动，虽然各个进程不能共享自己的内存空间，但是系统的内核空间是共享的，每个进程都可以访问，而Binder Driver即存在于内核空间，BBinder和BpBinder都是通过它来通信的，所以可以把Binder驱动当作是BBinder和BpBinder之间的一座桥梁。当然BBinder和BpBinder也不是直接和Binder驱动打交道，它们中间还隔着一个IPCThreadState。每一个进程都有一个ProcessState对象，它负责打开Binder驱动，这样该进程中的所有线程都可以通过Binder驱动通信，而每个线程都会有一个IPCThreadState对象，它才是真正读写Binder驱动的主角，它通过talkWithDriver()和驱动打交道，将IPC数据写入驱动中。

### Java-->C++对象转换

回到我们上面的Demo，客户端连接成功后拿到了BinderProxy对象，那么这个服务端的Binder对象是如何转为BinderPrxoy的呢？这点我们要看bindService的源码，bindService流程很长，感兴趣的读者可以自己去看后者直接看老罗的分析，我这里直接告诉读者大致过程：

**1.客户端请求bindService，先会请求ActivityManagerService;**

**2.ActvityManagerService再去找到对应的Service，让Service所在进程创建并启动Service;**

**3.Service调用AMS.publishService()将Binder对象传递给AMS;**

**4.AMS拿到的Binder对象同样为BinderProxy对象，然后调用 c.conn.connected(r.name, service)方法，将BinderProxy对象传递给客户端。**

这里要知道的是，AMS也是处在一个单独的进程中，所以Binder对象不是直接返回给AMS，AMS也不是直接返回给客户端的，而是经过了Binder驱动。服务端Service将Binder对象传递给AMS时，会调用AMS在服务端的代理对象ActivityManagerProxy.publishService()方法，准确的说，Service端在此时成了AMS的客户端。
	
	public void publishService(IBinder token,
            Intent intent, IBinder service) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(service);
        mRemote.transact(PUBLISH_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
该方法中通过Parcel.writeStrongBinder()，将服务端的Binder对象转化成C++中的flat\_binder\_object对象，并将Binder对象的地址赋值给flat\_binder\_object对象中的cookie。

	struct flat_binder_object {
          unsigned long type;
          unsigned long flags;
          union {
                  void *binder; /* local object */
                  signed long handle; /* remote object */
          };
          void *cookie;
    };
flat\_binder\_object对象最终被包装在了data中，然后通过mRemote.transact(),将data数据传送到IPCThreadState, IPCThreadState再次将data包装成binder\_transaction\_data，并调用talkWithDriver(),将包装好的数据传递给Binder驱动。

**Binder驱动收到数据后，并不会急着将数据传给AMS，因为传送的数据中有flat\_binder\_object，所以它会查询flat\_binder\_object对应的Binder对象在驱动中是否有Binder节点，如果没有，则会利用flat\_binder\_object中的cookie值(指向BBinder)创建一个Binder节点，并为该节点生成一个索引值，将索引值赋给flat\_binder\_object的handle。**

然后AMS就可以取数据了，AMS同样有一个IPCThreadState对象，Binder驱动将数据传递给该对象，接收端拿到了数据后，会调用Parcel.readStrongBinder()，这在java层是一个jni方法，该方法会先调用Native层的readStrongBinder()，利用flat\_binder\_object中的handle值创建BpBinder对象，然后该Native方法返回BpBinder的指针，jni方法再根据BpBinder指针创建java层的BinderProxy对象并返回给java层。至此BpBinder和BinderProxy对象都已经创建完毕。

### BpBinder与BBinder的通信
上面我们已经拿到了BpBinder，那BpBinder又将如何与BBinder通信呢？这里不得不提一个重要的变量：**handle**。还记得在创建BpBiner的时候，需要传入flat\_binder\_object的handle吗，而这个handle是Binder节点在驱动中的索引，即位置。这样当BpBinder通过transact()调用BBinder时，Binder驱动就可以根据BpBinder提供的handle值找到Binder在驱动中的节点，Binder节点保存了BBinder的地址，从而找到了BBinder，实现对BBinder的调用。

### 系统服务的注册

上面的例子展示的**匿名Binder**的通信，为什么说是匿名，因为Binder对象并没有在**ContextManager**中实名注册。

ContextManager又是什么？它是系统服务的管理者，它在java层和C++层的代理对象都为**ServiceManager**，系统服务可以通过**ServiceManager.addService()**将自己实名注册到ContextManager中。为什么要实名注册？如果不采用实名的方法，系统服务那么多，你难道一个个的去记得它的handle？

ServiceManager.addService()会将服务的Binder对象传到Binder Driver，Binder Driver为其生成Binder节点后，ContextManager就会将服务的实名和其Binder节点的索引handle保存到自身。当客户端需要找一个系统服务时，只需将服务名**ServiceManager.getService()**，ServiceManager就可以找到服务的索引handle，并创建对应的BpBinder对象，从而建立通信。比如java层的ActivityManagerService就是一个Binder类，我们就可以通过**ServiceManager.getService("activity")**，拿到它的代理对象，从而进行Activity生命周期的调度。

上面说到ServiceManager是ContextManager的代理，因为ContextManager本身也就是一个服务，服务端和客户端想调用它的addService或getService时，也必须通过ServiceManager来跨进程。可是我该怎么拿到这个代理呢？我总不能调用它的getService来获取它自己吧。Binder这一点设计的很奇妙，ContextManager在BinderDriver中的节点索引为0，谁让它是老大呢。这样大家都知道了，我想调用ContextManger，直接根据handle＝0就可以生成它的代理对象BpBinder了，从而创建代理对象ServiceManager。

## 总结
系统中每个app和系统服务都活在自己的进程中，无法访问彼此的内存空间，正是因为相互通信的强烈需求，从而诞生了Binder。Binder驱动就像所有进程共同的地盘，系统服务可以这里留下自己的地址，而客户端可以可以根据这些地址找到对应的服务，相互通信，从而千里姻缘一线牵，百年恩爱双心结。
