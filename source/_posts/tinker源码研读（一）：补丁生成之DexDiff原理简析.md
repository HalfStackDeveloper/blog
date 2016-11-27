---
title: tinker源码研读（一）：补丁生成之DexDiff原理简析
date: 2016-10-19 11:45:50
tags:
---

## 前言
微信的热修复框架Tinker已经在国庆节之前开源了，成为了github.com/Tecent下第一个项目，刷爆了各位开发者的朋友圈。作为一个超级APP的HotFix库，Tinker不仅值得我们**compile**，更值得我们**read**。

<!-- more -->

## 原理
Tinker和以往的HotFix库思路不太一样，它更像是APP的增量更新，在服务器端通过差异性算法，计算出新旧dex之间的差异包，推送到客户端，进行合成。传统的差异性算法有BsDiff，而Tinker的牛逼之处就在于它自己基于Dex的文件格式，研发出了DexDiff算法。

## 补丁生成之DexDiff
补丁生成是在编译阶段进行的，当我们在build.gradle中配置好tinkerOldApkPath后，调用tinkerPatchDebug，补丁就开始生成了。

调用链为：

TinkerPatchSchemaTask.tinkerPatch()  
->Runner.gradleRun()  
->Runner.run()  
->Runner.tinkerPatch()  
->ApkDecoder.patch()   
......  
->DexPatchGenerator.executeAndSaveTo()


executeAndSaveTo()方法是DexDiff算法的真正入口，DexDiff算法的特点在于它深入分析了Dex文件格式，深度利用Dex的格式来减少差异大小。

在了解DexDiff之前，我们需要对Dex文件有个初步了解，Dex的文件格式如下表。参考自这里：[Dalvik可执行格式（dex）上](http://www.cnblogs.com/cncooper/p/4361819.html)

| 数据名称      | 解释         
| ------------- |:-------------:|
| header     |dex文件头部，记录整个dex文件的相关属性|
| string_ids |字符串标识列表，涉及了文件中所用到的所有字符串，包括内部名称（比如类型描述符）或者代码引用的常量对象。这个列表根据字符串内容按照UTF-16（避免了各地区语言差异导致的不同）来进行排序，不得包含重复条目|
| type_ids   |类型标识列表，涉及了文件中所有用到的类型（如类、数组或者原始类型），不论是否是在文件中定义。这个列表必须按照类型字符串在string_ids中的索引来排序，不得含有重复条目。|
| proto_ids  |方法原型标识列表涉及了文件中所有用到的方法原型。列表按返回值类型在type_ids中的索引进行排序，索引相同的话再按参数类型在type_ids中的索引排序，不得含有重复条目。|
| field_ids  |类属性（类成员）标识列表，涉及了文件中所有用到的类属性，不论其是否在文件中定义。列表依次按照所在类的类型（按type_ids索引）、属性名（按string_ids索引）、自身类型（按type_ids索引）进行排序，不得含有重复条目。|
| method_ids |方法标识列表，涉及了文件中所有用到的方法，不论是否在文件中定义。列表依次按照方法所在类的类型（按type_ids索引）、方法名（按string_ids索引）、方法原型（按proto_ids索引），不得含有重复条目。|
| class_defs |类定义列表，列表的顺序必须符合一个类的基类以及其所实现的接口在这个类的前面这一规则。此外，列表中出现多个同名类的定义是无效的。|
| data       |数据区，包含上述各个结构所需的所有支持数据。不同的条目有不同的数据对齐要求，如果有需要，会在条目之前插入若干字节以满足合适的对齐|
| link_data  |用于静态链接文件的数据，数据的类型其实是不确定的，对于不存在链接的文件，这个区域是空的，此外不同的运行时实现也会对这一区域的数据格式做相应修改。|

想要深入了解Dex文件格式的可以看Google官方文档：[https://source.android.com/devices/tech/dalvik/dex-format.html](https://source.android.com/devices/tech/dalvik/dex-format.html)，《Android软件安全与逆向分析》这本书里关于Dex文件也有很详细的讲解。

了解了Dex的格式后，让我们来看一下DexDiff的基本步骤。

**DexDiff的主要步骤如下：**

**Step1:**计算出new dex中每项Section的大小，比如string_ids在dex文件中所占大小。

	int patchedStringIdsSize = newDex.getTableOfContents().stringIds.size * SizeOf.STRING_ID_ITEM;

**step2:**根据表中前一项的偏移地址和大小，计算出每项Section的偏移地址。

	this.patchedStringIdsOffset = patchedHeaderOffset + patchedheaderSize;

**step3:**调用DexSectionDiffAlgorithm.execute()，将new dex与old dex中的每项section进行对比，对于每项Section，遍历其每一项Item，进行新旧对比，记录ADD，DEL标识，存放于patchOperationList中。接着遍历patchOperationList，添加REPLACE标识，最后将ADD，DEL，REPLACE操作分别记录到各自的List中。

这里面的新旧对比的算法很有趣，它是直接将oldItem.compareTo(newItem)，结果小于0记为DEL，大于0记为ADD。为什么可以直接这样比较呢？别忘啦，从上面Dex文件格式表中可知，Dex文件中的Section都是经过排序的。

![](http://od35ecbnl.bkt.clouddn.com/tinker.png)

上图中，old dex中下标为2的item "c" 被DEL，c后面的元素前移，new dex中下标5处ADD一个item "f"，而在下标8处又ADD一个item "j"。
  
下标5的"f"在这里被记为ADD，但是它其实是REPLACE了"g"。在之后遍历patchOperationList时，算法会判断operation前一个opearation（prevPatchOperation）的类型，若prevPatchOperation为DEL，而自己刚好为ADD，那么就将自己的类型改为REPLACE。

	Iterator<PatchOperation<T>> patchOperationIt = this.patchOperationList.iterator();
	PatchOperation<T> prevPatchOperation = null;
	while (patchOperationIt.hasNext()) {
       PatchOperation<T> patchOperation = patchOperationIt.next();
       if (prevPatchOperation != null
           && prevPatchOperation.op == PatchOperation.OP_DEL
           && patchOperation.op == PatchOperation.OP_ADD
          ) {
              if (prevPatchOperation.index == patchOperation.index) {
                  prevPatchOperation.op = PatchOperation.OP_REPLACE;
                  prevPatchOperation.newItem = patchOperation.newItem;
                  patchOperationIt.remove();
                  prevPatchOperation = null;
                } else {
                  prevPatchOperation = patchOperation;
                }
            } else {
                prevPatchOperation = patchOperation;
            }
     }

**step4:**调用DexPatchGenerator.writePatchOperations()，将记录写入补丁。  
对于DEL：
开辟一块DelOpIndexList大小的区域(DelOpIndexList中记录了每块要删除部分的索引),遍历记录DelOpIndexList，对于每一个DEL索引，计算出其相对于前一个DEL索引的偏移，记录偏移量。

	buffer.writeUleb128(delOpIndexList.size());
        int lastIndex = 0;
        for (Integer index : delOpIndexList) {
            buffer.writeSleb128(index - lastIndex);
            lastIndex = index;
        }
对于ADD和REPLACE，也会进行和DEL一样的操作。

	buffer.writeUleb128(addOpIndexList.size());
        lastIndex = 0;
        for (Integer index : addOpIndexList) {
            buffer.writeSleb128(index - lastIndex);
            lastIndex = index;
        }
        
    buffer.writeUleb128(replaceOpIndexList.size());
        lastIndex = 0;
        for (Integer index : replaceOpIndexList) {
            buffer.writeSleb128(index - lastIndex);
            lastIndex = index;
        }

不同的一点是，ADD和REPALACE还会接着写入新增和替换的item。

	 for (T newItem : newItemList) {
            if (newItem instanceof StringData) {
                buffer.writeStringData((StringData) newItem);
            } else
            if (newItem instanceof Integer) {
                // TypeId item.
                buffer.writeInt((Integer) newItem);
            } else
            if (newItem instanceof TypeList) {
                buffer.writeTypeList((TypeList) newItem);
            }
            ......
    }

## 总结

这里只是简单的介绍了DexDiff的简要过程，中间其实省去了很多细节，而这些细节与Dex文件格式联系非常紧密，想要彻底的了解DexDiff原理，还需好好研究Dex文件。

(转载请注明ID:**半栈工程师**，欢迎访问**个人博客**：[https://halfstackdeveloper.github.io/](https://halfstackdeveloper.github.io/))