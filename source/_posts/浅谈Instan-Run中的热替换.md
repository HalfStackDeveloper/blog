---
title: 浅谈Instan Run中的热替换
date: 2016-08-10 15:02:08
tags:
---

## 前言:
自从Android Studio 2.0发布以来，相信广大的攻城狮朋友们都已经用上了**Instant Run**这个新特性，还没用上的朋友们，赶紧去Google官网了解一下吧 https://developer.android.com/studio/run/index.html#instant-run

<!-- more -->

<font size="3" color="#000000">**Instant Run主要分为三种方式来加载app:**</font>

<font size="2" color="#4d4d4d">**Hot Swap：**</font>
这是最令人激动的方式，它可以在不重启Activity的情况下实现代码的替换，简直是逆天啊！但是热替换的条件很苛刻，只能是在简单的修改了代码的情况下，AS才会采用这种方式。

<font size="2" color="#4d4d4d">**Warm Swap：**</font>
暖替换，是对热替换的让步，它会重启你所修改的Activity，但是不会重启App。如果在项目中修改了资源，AS会自动选择这种方式。

<font size="2" color="#4d4d4d">**Cold Swap:**</font>
如果你改变了代码的结构，如继承和改变了方法名，那么AS也只能无奈的选择冷替换了，它会重启整个App。

## 探索:
接下来让我们来探索一下神奇的Hot Swap。

这是一个很简单的Activity，就用它来窥探Instant Run吧。
		
	public class MainActivity extends AppCompatActivity {
    	private Button mBtnTest;
    	private int mNum;

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_main);
           	mBtnTest = (Button) findViewById(R.id.btn_test);
        	setListener();
    	}

    	private void setListener() {
        	mBtnTest.setOnClickListener(new View.OnClickListener() {
           	 	@Override
            	public void onClick(View v) {
                	mNum++;
                	Log.e("InstantRun", "Num: " + mNum);
        	});
    	}
    }
    
    
点一下按钮，打出如下log：

	08-11 16:51:16.730 4125-4125/com.wangxiandeng.instantruntest E/InstantRun: Num: 1

再点一下：

	08-11 16:54:36.300 4125-4125/com.wangxiandeng.instantruntest E/InstantRun: Num: 2

你们看到现在，是不是心想，你特么在逗我么？别急，接着往下走。

把代码修改为：

	Log.e("InstantRun", "Num: " + mNum*2);

点击Instan Run 闪电按钮⚡️，Activty没有重启，这时候再点击按钮

	08-11 17:02:15.340 14022-14022/com.wangxiandeng.instantruntest E/InstantRun: Num: 6
	
可见，mNum的值在Hot Swap时并没有重置，而是保持了之前的值：2，也就是说，Activity的所有生命周期方法并没有重走一遍，但是现在log打印出来为6，所以代码确实被替换了，那Hot Swap究竟是怎么做到这一点的呢，让我们来揭开它的神秘面纱。

   
## 原理:
Instant Run其实类似于这两年很火的Hotfix，根据Instant Run的思想，甚至可以自己去鼓捣出一个Hotfix库。

Hot Swap 看起来很高大上，其实玩的就是狸猫换太子的把戏。在app的第一次编译阶段，它利用transform 在我们的每一个类里注入了一个变量：$change，这是一个IncrementalChange类型的变量。各位看官想必又要骂我了：你说注入就注入了啊？

那我们回到刚才那个Activity，证明它被注入了$change字段。

现在修改onClick中的代码如下：

                Class clazz = MainActivity.class;
                try {
                    Field changeField = clazz.getDeclaredField("$change");
                    changeField.setAccessible(true);
                    Object changeValue = changeField.get(this);
                    Class changeClass = changeValue.getClass();
                } catch (Exception e) {
                    e.printStackTrace();
                } 
                
  再次点击Activity中按钮，log打印为：
                
 	08-11 17:25:15.830 3311-3311/com.wangxiandeng.instantruntest E/InstantRun: Class: class com.wangxiandeng.instantruntest.MainActivity$override 
 	
 事实证明，Activity中确实有$change这个变量，细心的读者还会发现，这个$change 变量的运行类型为 com.wangxiandeng.instantruntest.MainActivity$override  
 
 这里的MainActivity$override其实就是狸猫，也就是我们经常说的补丁，它实现了IncrementalChange接口，并且重写了MainActivity中的所有方法。我们在onClick中再加一句代码
 
	printMethods(changeClass);
				
printMethods会打印出MainActivity$override中的所有方法。

	public static void printMethods(Class cl) {
        Method[] methods = cl.getDeclaredMethods();
        for (Method m : methods) {
            Class retType = m.getReturnType();
            String name = m.getName();
            System.out.print("  ");
            String modifiers = Modifier.toString(m.getModifiers());
            if (modifiers.length() > 0) {
                System.out.print(modifiers + " ");
            }
            System.out.print(retType.getName() + " " + name + "(");
            Class[] paramTypes = m.getParameterTypes();
            for (int j = 0; j < paramTypes.length; j++) {
                System.out.print(paramTypes[j].getName());
                if (j < paramTypes.length - 1) {
                    System.out.print(", ");
                }
            }
            System.out.println(");");
        }
    }
    
点击Activity中按钮，打印出方法如下：

	public transient java.lang.Object access$dispatch(java.lang.String, [Ljava.lang.Object;);
		
	public static java.lang.Object init$args([Lcom.wangxiandeng.instantruntest.MainActivity;, [Ljava.lang.Object;);
	
	public static void init$body(com.wangxiandeng.instantruntest.MainActivity, [Ljava.lang.Object;);
	
	public static void onCreate(com.wangxiandeng.instantruntest.MainActivity, android.os.Bundle);
	
	public static void printMethods(java.lang.Class);
	
	public static void setListener(com.wangxiandeng.instantruntest.MainActivity);
	
从Log中可以看出，MainActivity$override中包含了MainActivity中所有的方法，包括onCreate(),  printMethods(),  setListener()。

看到这里，聪明的读者应该已经猜测出Instan Run的原理了，其实也就是和代理差不多，MainActivity在执行方法时，会先判断它的代理（$change）是否为空，如果不为空，就执行代理里的方法。这样当我们修改了某个类方法里的代码，AS会自动的创建一个该类的代理（xx$override),并将代理赋值给该类的$chang字段，这样我们的修改在不重启Activity的情况下也能生效了。

代理类是通过access$dispatch()方法来进行函数分发的，传入的参数为所要执行方法的签名和参数，access$dispatch()会根据方法签名的hashcode寻找到目标方法，并传入参数执行。接下来我们再来试验一下。

在MainActivity中再添加一个方法：

	private void sayHello(String text) {
        Log.e("InstantRun", text);
    }
    
接着在onClick try块中再添加两行代码，通过反射MainActivity$override 中的access$dispatch()方法，实现调用补丁中的sayHello()。

	Method dispatchMethod = changeClass.getDeclaredMethod("access$dispatch", new Class[]{String.class, Object[].class});
    dispatchMethod.invoke(changeValue, "sayHello.(Ljava/lang/String;)V", new Object[]{MainActivity.this, "Hello World!"});
    
在第二行代码中，我们将sayHello()的方法签名以及一个“Hello World!”字符串传入给access$dispatch方法，接下来看看能不能成功的调用sayHello()。

	08-11 17:25:15.840 3311-3311/com.wangxiandeng.instantruntest E/InstantRun: Hello World!
	
Log中成功的打印出了Hello World! 

到这里，大家应该对Instan Run Hot Swap的来龙去脉有所了解了，那么补丁文件又是怎么加载进来的呢？
当我们修改代码，并点击运行按钮时，AS会创建一个AppPatchesLoaderImpl，该类中记录了哪些类被修改了，然后通过scoket，将补丁文件和AppPatchesLoaderImpl发送到设备，调用设备的
handleHotSwapPatch()方法。
	
	private int handleHotSwapPatch(int updateMode, @NonNull ApplicationPatch patch) {
       try {
           String dexFile = FileManager.writeTempDexFile(patch.getBytes());
           String nativeLibraryPath = FileManager.getNativeLibraryFolder().getPath();
           DexClassLoader dexClassLoader = new DexClassLoader(dexFile,
                   mApplication.getCacheDir().getPath(), nativeLibraryPath,
                   getClass().getClassLoader());
           // we should transform this process with an interface/impl
           Class<?> aClass = Class.forName(
                   "com.android.tools.fd.runtime.AppPatchesLoaderImpl", true, dexClassLoader);
           try {
               PatchesLoader loader = (PatchesLoader) aClass.newInstance();
               String[] getPatchedClasses = (String[]) aClass
                       .getDeclaredMethod("getPatchedClasses").invoke(loader);
               if (!loader.load()) {
                   updateMode = UPDATE_MODE_COLD_SWAP;
               }
           } catch (Exception e) {
               updateMode = UPDATE_MODE_COLD_SWAP;
           }
       } catch (Throwable e) {
           updateMode = UPDATE_MODE_COLD_SWAP;
       }
       return updateMode;
  	 }
  	
该方法首先新建了一个ClassLoader，将补丁记录类AppPatchesLoaderImpl加载进来，然后调用AppPatchesLoaderImpl的load方法，load()方法中会遍历并记载所有的补丁类，并反射原有类的$change变量，赋值以补丁类。

想深入了解补丁加载的同学，可以看一看w4lle's Notes的文章[《从Instant run谈Android替换Application和动态加载机制》](http://w4lle.github.io/2016/05/02/从Instant%20run谈Android替换Application和动态加载机制/)。

## 总结:

至此为止，Instan Run中的Hot Swap基本流程已经讲完了，总的来说就是代理，有点类似支付宝的Andfix，不过Andfix是从jni层去修改方法指针，本质其实都是替换掉目标方法，运行补丁方法。








	




