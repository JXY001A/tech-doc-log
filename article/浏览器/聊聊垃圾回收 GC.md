<!--
 * @description: 
 * @author: JXY
 * @Date: 2019-10-29 17:01:10
 * @Email: JXY001a@aliyun.com
 * @LastEditTime: 2019-10-29 17:07:18
 -->

# 聊聊垃圾回收 GC

GC 全称 Garbage Collection ，即：垃圾回收。

## 计算机程序垃圾指的是什么呢？
写过计算机程序的都知道，程序中无时无刻伴不随着变量的引用，赋值，运算等操作。于是乎存在着某些变量在使用过后，程序不会再用到它们，但是他们依然占据着一定的内存空间。内存中这样不会被程序再次使用的数据便可称之为‘垃圾’。

## 什么是回收？
简而言之，回收就是将上面讲到的 程序运行时产生的‘垃圾’ 释放掉，以便于程序再次使用这块内存区域。

## GC 的历史

- 1960 年 John McCarthy  首次发布著名的CG 算法 ，**GC 标记-清除算法**
- 1960 年 George E. Collins  发布了**引用计数**的GC算法
- 1963 年 Marvin L. Minsky 发布了 **GC 赋值算法**

令人惊叹的是，目前所有的GC算法只不过是上述3种算法的组合或应用，也就是说从1963年赋值算法诞生时，到目前为止没有诞生新的GC算法！

## GC 中的基本概念
首先我们明确个知识点，GC在内存中**销毁移动**的基本单位是对象。

### 对象
对象由头（head）和 域（field）构成。
![对象.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1570074031648-247ad928-6726-4c44-9987-da424009553f.png#align=left&display=inline&height=401&name=%E5%AF%B9%E8%B1%A1.png&originHeight=401&originWidth=487&search=&size=5393&status=done&width=487)
#### 头 head
对象中保存对象本身信息的部分称之为‘头’。head中的信息用户无法访问和修改，包含下列信息。

- 对象的大小
- 对象种类

#### 域 field
field 对于我们而言比较熟悉，在**JavaScript**使用属性操作符便可以直接访问的部分被称之为域。域包含两种数据类型。

- 指针 ，   即：引用数据类型
- 非指针， 即：基本数据类型，例如 true , false , 1, ……。

** **
### mutator 
指的是修改CG对象之间的引用关系。更准确的来讲它就是 ‘**应用程序**’本身，也就是我们写的代码。
**
### 堆 heap
堆指的是用于动态（也就是执行程序时）存放对象的内存空间。当 mutator 申请存放对象时， 所需的内存空间就会从这个堆中被分配给 mutator。

### 活动对象/非活动对像
也就是分配到内存空间中的对象中那些能够通过 **mutator** 引用的对象称之为 ‘活动对象’，反之为‘非活动对象’即：垃圾。

### 根 root
在CG世界中，根指的是对象指针的起点部分。也就是**mutator**(应用程序) 中的全局变量，通过递归这些根（**全局变量**）可以遍历到的对象就是活动对象，反之就是非活动对象（**垃圾**）。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1570076319845-f955e14f-2ab4-43f0-8912-599d8b13adae.png#align=left&display=inline&height=397&name=image.png&originHeight=397&originWidth=672&search=&size=22789&status=done&width=672)
（根与堆中对象的关系）


## GC标记-清除算法
如该算法的名称所描述，它的执行可以分为两个阶段，即：**标记** 和 **清除**。**标记**就是通过上面提到的**根(全局对象)**能遍历到的内存中的对象标记为**活动对象**，标记完成之后接下来就**清除**内存中没有被标记的对象，也就是**非活动对象。**通过这两个阶节阶段实现了内存空间的重复利用。
### 标记阶段
**标记过程**实际上可以通过伪代码来更加具体展现：

```
/*标记函数*/
mark_phase(){  
  // 遍历全局对象，即：根
  for(r : $roots)    
    	mark(*r) 
}

/* 标记遍历到的对象,然后继续遍历该对象的引用对象。 标记过程采用的是深度优先算法 */
mark(obj){
	// 遍历到已经设置为 true 的对象则不再处理
	if(obj.mark == FALSE)
		obj.mark = TRUE
    // 标记后继续遍历当前对象的引用对象，直至所有的活动对象都被遍历到
    for(child : children(obj))      
    	mark(*child) 
}
```

### 清除阶段
**清除阶段**的遍历过程是从堆的首地址开始，一个个的遍历对象的标志位。

伪代码如下所示：
```
sweep_phase(){
	//  $heap_start 指针指堆中的第一个对象的指针
  sweeping = $heap_start
  // 遍历堆时的边界控制，不能超出堆的大小
  while(sweeping < $heap_end)
    if(sweeping.mark == TRUE)
    	// 遇到标记为 true 的对象,则取消标志位，准备下一次GC
      sweeping.mark = FALSE
    else
    	// 这里就是关键的回收处理了，首先$free_list 就是‘空闲链表’，程序需要的内存就是从其中分配获得。
      
      // 可能有小伙伴看不懂这里，就详细解释一下：
      /*
      	 $free_list 是指向‘空闲链表’的指针，将它赋值给要回收对象的 next 域，那么接下来这个要被回收
         的对象就变成了‘空闲链表’的头，即：header,也就是这个对象被加入到了空闲链表中，接下来就会被
         当做空闲空间来分配给应用程序。最后再将头指针赋给 $free_list 变量，也就是 $free_list 回指向
         空闲链表，然后继续遍历……
      */
      sweeping.next = $free_list
      $free_list = sweeping
    
    // size 保存着被回收对象的大小，这步操作实际上就是移动至堆中的下一个对象，进行回收操作。   
    sweeping += sweeping.size
}

```

通过下图可以形象的看出完成一次 GC 回收过后堆的状态：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1572094415160-7e31c003-f194-4892-96c5-43ec71a482af.png#align=left&display=inline&height=625&name=image.png&originHeight=625&originWidth=1115&search=&size=47201&status=done&width=1115)
（一次GC回收过后堆的状态）




## 引用计数法
**GC** 被本质上来说就是一种研究如何**释放无法被引用对象**的技术。那么，可以想到，如果让对象自己记录一下，它有没有被程序引用。这就是**引用计数法**。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1572097565865-b6849417-1414-49e9-af03-4d8421a058b4.png#align=left&display=inline&height=156&name=image.png&originHeight=156&originWidth=319&search=&size=4649&status=done&width=319)
（引用计数法的对象）


**引用计数法**与 **mutator（应用程序）**的执行密切相关，也就是在程序处理数据对象的过程中通过增减计数器来实现内存管理。 在对象的生成和被引用时会发生计数器的增减，也就是 `new_obj()` 函数和 `update_ptr()` 函数。


### new_obj() 函数，分配内存
伪代码如下：
```
new_obj(size){
    // 从‘空闲链表’中为程序分配空间
	obj = pickup_chunk(size, $free_list)
    // 分配空间失败
    if(obj == NULL)    
        allocation_fail()  
    else
        // 为分配成功的对象设置计数器，并初始化
        obj.ref_cnt = 1    
        return obj 
 }
```


### update_ptr()函数，程序引用该对象
伪代码如下：
```
update_ptr(ptr, obj){
    // obj 对象的计数器增量操作
    inc_ref_cnt(obj)
    // ptr 指针原来执行的对象计数器减量操作
    dec_ref_cnt(*ptr)
    // ptr 指正指向 obj
    *ptr = obj
}

// 该函数就是让指针 ptr 要指向 obj 对象，实际中的代码类似于：
/*
	var a = {};
  var b = {};
  a 对象计数器自增，b 对象计数器自减，
  b=a;
*/
```

**inc_ref_cnt()** 函数伪代码实现：
```
// 这里很简单 ，就是计数器自增
inc_ref_cnt(obj){  
	obj.ref_cnt++ 
}
```

**dec_ref_cnt()** 函数伪代码实现：
```
dec_ref_cnt(obj){
	// obj 对象计数器自减
  obj.ref_cnt--
  // 如果计数器等于 0 ,也就是说 obj 对象变为垃圾了 
  if(obj.ref_cnt == 0)
  	// 那么被 obj 所引用的对象也应该做减量操作
    for(child : children(obj))      
    	dec_ref_cnt(*child)
    // 将 obj 对象连接至空闲链表
 		reclaim(obj) 
}
```

### 缺点
循环引用，造成内存无法被回收
```javascript
// 注意：例子仅用于描述场景，不符合真实环境情况

funciton Person(name,lover) { 
  this.name = name;
  this.lover= lover;
}

let jxy = new Person("jxy"); 
let xxx = new Person("xxx")
jxy.lover = xxx;
xxx.lover = jxy;

xxx = null;
jxy = null;

// 当GC采用引用计数法管理内存的时候，在上面例子中虽然变量都被赋值为空，但是两个对象本身确相互
// 引用，这样就导致了内存无法被有效回收，即：内存泄露
```

## GC复制算法
复制算法顾名思义，就是将堆中的所有活动对象复制到另外一个空间，然后原来的空间全部回收掉。这样的好处就是防止出现内存的碎片化，易于随后为程序分配新的空间。可以形象的理解为下图：
![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1572142786143-27cda528-d75e-4736-8d25-f822f24e6d53.png#align=left&display=inline&height=732&name=image.png&originHeight=732&originWidth=1284&search=&size=249231&status=done&width=1284)
### 复制算法
我们把原来的活动对象空间称之为 **From **空间，将要复制到的新空间称之为 **To** 空间。当 **From **空间被完全占满时，GC 会将活动对象复制到 **To** 空间。复制完成后，该算法会把 **From** 空间和 **To** 空间互换，本次 GC 也就结束了。GC 复制算法概要如下图所示：
![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1572143296634-9e456dd4-f809-450c-82a4-2a2c631d58a4.png#align=left&display=inline&height=723&name=image.png&originHeight=723&originWidth=1344&search=&size=97805&status=done&width=1344)
（GC 复制算法概要）


这里再说明一下，**mutator **就是应用程序本身，一次回收完成之后，程序会继续执行，再次产生垃圾，新的 **From** 空间会被填满，然后 GC 又开始新的一轮回收操作，回收操作伴随程序的整个生命周期。


下面我们通过伪代码来的看一下 GC 复制算法的具体实现思路。


**copying()** 函数伪代码：
```
copying(){	
	// 用 $free 变量记录 To 空间的开始位置
  $free = $to_start
  // 遍历所有的根对象，使用 copy 方法将他们复制到 To 空间
  for(r : $roots)
  	// 返回的 *r 是对象被复制到 To 空间后新的指针。 
    *r = copy(*r)
  // 复制完成之后，交换 From 和 To 空间
  swap($from_start, $to_start)
}
```

 **copy()**函数伪代码：
```
/* copy 函数将作为参数给出的对象复制，再递归复制其子对象 */
copy(obj){
	// obj 对象的 tag 域用于标记是否是一个已经被赋值过的对象 
  if(obj.tag != COPIED)
  	// 使用 copy_data 方法具体来拷贝 obj 对象，同时传入To 空间地址，以及 obj d对象的大小  
    copy_data($free, obj, obj.size)
    // 拷贝完成之后，标记一下，该对象已经被赋值过了. $free 变量指向新的 obj
    obj.tag = COPIED
    //  旧的 obj 对象的 forwarding 域保存新的 obj 对象的指针，用于后面将其赋给程序中原始的指向
    obj.forwarding = $free
    // $free 跳过已经被复制的 obj 的空间，指向 To 空间的空闲位置，方便下一次复制使用 
    $free += obj.size
    // 递归复制 obj 对象的引用对象
    for(child : children(obj.forwarding))
      *child = copy(*child)
  // 当拷贝完成之后返回新对象的指针
  return obj.forwarding
}
```

## Chrome V8 的垃圾回收
前面写了那么多，那么到底我们常用的浏览器使用的是那种回收算法呢？我想这可能是小伙伴们最关心的了。那么以我们最喜爱的 Chrome 为例，它使用的是多种回收算法的组合优化，而非某种单一算法。V8 的 GC 算法统称为**分代垃圾回收算法**，也就是通过记录对象的引用次数，将超过一定引用次数的对象划分为**老年对象**，剩下的称之为**新生代对象，**然后分别对他们采用不同到的垃圾回收算法**。**那这样划分到底有什么优势呢，我们知道程序中生成的大多数对象其实都是产生之后随即丢弃。以下面代码为例，函数内部生成了对象，在该函数执行完毕，出栈之后，包括函数本身以及它内部的变量都会立刻成为垃圾：

```javascript
// 该函数的执行上下文环境非全局作用域
function foo() {
	var a = {c:1};
  var c = {c:2};
}
```

那么对于这种新生代对象来说，回收就会变得很频繁，如果使用 **GC 标记清除算法**，那么就意味着每次清除过程需要处理很多的对象，会浪费大量的的时间。于是如果对新生代对象采用 **GC 复制算法**的只需要将活动对象复制出来，然后将整个 **From **清空即可，无需再去遍历需要清除的对象，达到优化的目的。而针对老年对象则不同，它们都有多个引用，也就意味着它们成为非活动对象的概率较小，也就可以理解为**老年对象**不会轻易变成垃圾。再进一步也就是**老对象**产生的垃圾很少，如果采用复制算法的话得不偿失，大量的**老年对象**被复制来复制去也会增加负担，所以针对老年对象采用的是**标记清除法，**需要清除的老年对象只是少数，这样**标记清除算法**会更有优势**。**还有随着程序的执行新生代的对象会变成老年对象，这个具体过程比较复杂，小的能力有限，这里也就一笔带过了。既然对象分为新生带对像和老年对象，那么它们在堆中是如何分布的呢，请看下图：
![image.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1572149930292-13db603f-c70b-4a30-932e-05f7a9f46a5d.png#align=left&display=inline&height=352&name=image.png&originHeight=483&originWidth=749&search=&size=53004&status=done&width=546)
（V8 的 VM 堆结构示意图 ）


这里我们只需要知道堆被分为新生带空间，和老年代空间即可。除了新生代空间中方的 **From **空间和 **To** 空间外，老年代空间中细分优化，各位大佬请自由探索，小的能有限，就不敢造次了o(╥﹏╥)o。


文章只是写一下自己学到理解的东西，有错误还望大佬们指出啊。^-^


## 文章参考

- [垃圾回收的算法与实现](https://www.ituring.com.cn/book/1460)
