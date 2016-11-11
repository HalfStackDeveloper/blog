---
title: 基于Instant Run思想的HotFix方案实现
date: 2016-09-23 22:49:48
tags:
---
## 前言
近一年来，各种HotFix库层出不穷，各家大厂百花齐放，QQ空间最早提出了自己的热修复实现，接着阿里也开源了自家的AndFix（貌似阿里百川已经给开发者提供了新的Hotfix功能），现在微信又有了Tinker，各家都如此关心HotFix，无非是线上版本的bug对产品影响太大，尤其是DAU比较高的app，更是不能容忍。前几天看到美团基于Instant run原理推出了自己的Hotfix库，不过貌似没有开源，于是自己就按照Instant run的原理也鼓捣出了一个简单的HotFix实现，**可以在不重启App和Activity的条件下实现修复，**代码地址会在文章最后贴出，供大家研究学习。

<!-- more -->

## 实现效果
先让大家看看具体的实现效果是怎样的，很简单，一个Acitivty中点击按钮，会弹出Toast“我有bug！”，然后加载补丁，再点击按钮，会弹出“我是补丁，没有bug啦!”

![](http://od35ecbnl.bkt.clouddn.com/hotfix.gif)

接下来让我们看看怎么实现这个库。

## Instant Run原理

Instant run的原理是采用了狸猫换太子的戏法，在编译阶段给每个类都注入了一个$change(代理，即补丁)变量，并且在每个方法前都注入了一段代码，判断$change是否为空，如果不为空，就执行代理里的方法。

关于Instant Run具体的原理，我在文章[《浅谈Instan-Run中的热替换》](https://halfstackdeveloper.github.io/2016/08/10/浅谈Instan-Run中的热替换/)中已经讲了很多，这里不再赘述，建议不了解的同学在阅读本文前先看看这篇文章。

## 实现

### Step1:代码注入
上面说到Instan Run在编译期间给每个类都注入了变量和代码，那么这是怎么实现的呢？其实很简单，android studio给我们提供了transform Api，transform其实也就是打包过程中的一个task，我们可以根据这个特性，利用javassist来注入代码。关于代码注入这块，我参考了文章[《通过自定义Gradle插件修改编译后的class文件》](http://www.jianshu.com/p/417589a561da),谢谢这位同学慷慨的分享，具体过程大家可以看看这篇文章，我就不讲详细的步骤了。

代码注入的实现代码如下：

	 File dir = new File(path)
        if (dir.isDirectory()) {
            dir.eachFileRecurse { File file ->

                String filePath = file.absolutePath
                //确保当前文件是class文件，并且不是系统自动生成的class文件
                if (filePath.endsWith(".class")
                        && !filePath.contains('R$')
                        && !filePath.contains('R.class')
                        && !filePath.contains("BuildConfig.class")
                        && !filePath.contains("\$Patch.class")
                        && !filePath.contains("PatchBox.class")) {
                    // 判断当前目录是否是在我们的应用包里面
                    int index = filePath.indexOf(packageName);
                    boolean isMyPackage = index != -1;
                    if (isMyPackage) {
                        int end = filePath.length() - 6 // .class = 6
                        String className = filePath.substring(index, end).replace('\\', '.').replace('/', '.')
                        //开始修改class文件
                        CtClass c = pool.getCtClass(className)

                        if (c.isFrozen()) {
                            c.defrost()
                        }
                        pool.importPackage("com.wangxiandeng.savior")

                        //给类添加$savior变量，即补丁变量
                        CtField savior = new CtField(pool.get("com.wangxiandeng.savior.Savior"), "\$savior", c);
                        savior.setModifiers(Modifier.STATIC);
                        c.addField(savior);

                        //遍历类的所有方法
                        CtMethod[] methods = c.getDeclaredMethods();
                        for (CtMethod method : methods) {
                            //在每个方法之前插入判断语句，判断类的补丁实例是否存在
                            StringBuilder injectStr = new StringBuilder();
                            injectStr.append("if(\$savior!=null){\n")
                            String javaThis = "null,"
                            if (!Modifier.isStatic(method.getModifiers())) {
                                javaThis = "this,"
                            }
                            String runStr = "\$savior.dispatchMethod(" + javaThis + "\"" + method.getName() + "." + method.getSignature() + "\" ,\$args)"
                            injectStr.append(addReturnStr(method, runStr))
                            injectStr.append("}")
                            print("插入了：" + injectStr.toString() + "语句")
                            method.insertBefore(injectStr.toString())
                        }
                        c.writeFile(path)
                        c.detach()
                    }
                }
            }
        }
        
上面这段代码中，我们给每个类都注入了一个类型为Savior的静态变量$savior，并且在每个方法前加入了一段代码，判断$savior是否为null，如果不为null，则执行$savior.dispatchMethod()，传入方法的方法签名和参数，让补丁类代以执行。


### Step2:制作补丁类

补丁类的命名方式必须为XXX$Patch,比如MainActivity有bug，那么就制作一个名为MainActivity$Patch的补丁类，注意补丁类必须和原有类要放在同一包下。

先让我们写一个MainActivity，该类有bug（当然不是真的有bug啊）

	public class MainActivity extends AppCompatActivity {

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_main);
        	
        	//将补丁文件从资源目录拷贝到sd卡
        	FileUtil.copyJarToFile(this);
        	
        	findViewById(R.id.btn_show).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View v) {
                	show();
            	}
        	});
			//点击加载补丁
        	findViewById(R.id.btn_load).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View v) {
                	try {
           				PatchLoader.getInstance().loadPatch(PatchUtil.PATCH_PATH);
                    	Toast.makeText(MainActivity.this, "load success", Toast.LENGTH_SHORT).show();
                	} catch (Exception e) {
                    	e.printStackTrace();
                	}
            	}
        	});
    	}

    	public void show() {
        	Toast.makeText(this, "我有bug!", Toast.LENGTH_SHORT).show();
    	}
	}

现在我们要制作一个补丁类，用来修复bug。如果原有类继承自某个类，则补丁类同样要继承自该类，并且要实现Savior接口。

	public class MainActivity$Patch extends Activity implements Savior{

    	@Override
    	public Object dispatchMethod(Object host, String methodSign, Object[] params) {
        	MainActivity mainActivity = (MainActivity) host;
        	switch (methodSign.hashCode()) {
            	case -641568046:
                	onCreate(mainActivity, (Bundle) (params[0]));
                	break;
            	case -340027132:
                	show(mainActivity);
                	break;
        	}

        	return null;
    	}

    	protected void onCreate(final MainActivity host, Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_main);
        	findViewById(R.id.btn_show).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View v) {
                	show(host);
            	}
        	});

        	findViewById(R.id.btn_load).setOnClickListener(new View.OnClickListener() {
            	@Override
            	public void onClick(View v) {
                	try {
                    	PatchLoader.getInstance().loadPatch(getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS).getPath() + "/patch.dex");
                    	Toast.makeText(host, "load success", Toast.LENGTH_SHORT).show();
                	} catch (Exception e) {
                    	e.printStackTrace();
                	}
            	}
        	});
    	}

    	public void show(MainActivity host) {
        	Toast.makeText(host, "我是补丁，没有bug啦！", Toast.LENGTH_SHORT).show();
    	}
	}

补丁类需要实现原有类的所有方法，并且要重写原有类有bug的方法，在该例子中，原有类的show()方法含有bug，所以重写了show()方法，当然其他方法可能也会视情况有一些变动。补丁类需要重写接口中dispatchMethod()方法，根据方法签名的hashcode来进行具体的方法调用。

这里还有一个我未解决的问题，即在补丁类中调用原有类的super()方法，Instant run采用的是在每个原有类里又添加了一个函数access$super(),用来调用super方法，这样补丁类遇到super方法时，直接调用原有类的access$super()方法即可，不过就像美团在文章中所说的，该方法会增加app的方法数，所以不采用。美团采用的方法是修改super方法的调用指令。

我们来用javap -c 命令看一下MainActivity的字节码
其中调用super.onCreate(）的指令为：

	 30: invokespecial #2                  // Method android/support/v7/app/AppCompatActivity.onCreate:(Landroid/os/Bundle;)V

可以看见调用的是invokespecial指令（不知美团为何说是invokesuper），该指令用于调用实例构造器方法，私有方法和父类方法。美团的意思应该是去修改指令为invokespecial，不知是不是使用java bytecode editor手动操作，如果有同学知道，希望留言中告知。

接着写一个补丁记录类，用来记录有哪些补丁

	public class PatchBox implements IPatchBox {
    	@Override
    	public List<String> getPatchClasses() {
        	List<String> list = new ArrayList<>();
        	list.add("com.wangxiandeng.saviortest.MainActivity$Patch");
        	return list;
    	}
	}
	
### Step3:补丁打包

补丁写完后，需要打包成dex，首先在编译过程，拷贝出补丁类和PatchBox.class文件，依据补丁类所在包名，放在文件夹下，比如新建一个总文件夹patch，再新建一个
com/wangxiandeng/saviortest/文件夹，放入MainActivity$Patch.class和PatchBox.class,**然后按照以下步骤操作：**

1.cd 到patch目录；

2.利用jar cvf patch.jar * 指令打包成jar文件。

3.利用build-tools目录下的dx指令：dx --dex --output=patch_dex.jar patch.jar指令，打包成dex的jar包，patch_dex.jar即为我们打包好的补丁。

### Step4:补丁加载

将补丁放在sd卡中，执行补丁加载过程。补丁加载的核心代码如下：

	//加载补丁Dex文件
    DexClassLoader dexClassLoader = new DexClassLoader(patchPath, getOdexPath(), null, getClass().getClassLoader());

    //加载补丁装载类PatchBox
    Class<?> patchBoxClass = Class.forName(mPatchBoxName, true, dexClassLoader);
    IPatchBox patchBox = (IPatchBox) patchBoxClass.newInstance();

    //遍历加载补丁类
    for (String className : patchBox.getPatchClasses()) {
        Class<?> patchClass = dexClassLoader.loadClass(className);
        Object patchInstance = patchClass.newInstance();

        //反射修改bug类的mSavior字段
        int index = className.indexOf("$Patch");
        if (index == -1) {
            Log.e("Savior:", "incorrect name for patch, please rename your patch according to the README.md");
            return;
        }
        String bugClassName = className.substring(0, index);
        Class<?> bugClass = getClass().getClassLoader().loadClass(bugClassName);
        Field saviorField = bugClass.getDeclaredField("$savior");
        saviorField.setAccessible(true);
        saviorField.set(null, patchInstance);
    }
    
**补丁加载过程主要分为3步：**

1.利用DexClassLoader加载补丁；

2.加载补丁记录类PatchBox；

3.遍历PatchBox中记录的补丁类并实例化，反射对应原有类的$savior字段，赋值为补丁实例。

**至此，补丁类就已经加载完毕，此时调用原有类的bug方法，实际上调用的是补丁类的修复方法。**


## 结论

以上就是一个简单的HotFix库，当然这只供大家学习使用，离商用还差的很远，真正的HotFix库要考虑的地方还有很多，毕竟是用来修复bug的，总不能库自身一堆bug吧。不过Instant Run确实是实现HotFix的一个很好的方案，不需要考虑android版本兼容性的问题，还可以实现修复的即时生效，期待美团Hotfix库的开源！

代码地址：[https://github.com/HalfStackDeveloper/Savior](https://github.com/HalfStackDeveloper/Savior)