---
layout: post
title: "javascript模块化开发的规范是个啥意思"
description: "Javascript模块化开发规范是个啥意思"
tags: [javascript,amd,cmd,seajs,commonjs,requirejs]
date: 2015-03-13 10:48:51
---

* [WHY WEB MODULES](http://requirejs.org/docs/why.html)
* [AMD规范](https://github.com/amdjs/amdjs-api/wiki/AMD)
* [CMD规范](https://github.com/seajs/seajs/issues/242)
* [AMD vs CMD: 玉伯在知乎的回答](http://www.zhihu.com/question/20351507/answer/14859415)
* [使用 AMD、CommonJS 及 ES Harmony 编写模块化的 JavaScript](http://justineo.github.io/singles/writing-modular-js/)

---

不管什么系统，变复杂以后，为了更高效地构建、维护，都会细化成各个相对独立的模块，建筑有模块化建筑，
家具有宜家，玩具也有乐高积木，简单的模块可以更快捷地组装成型。

话归正传，前端功能越丰富强大，也要走模块化开发的道路，各个模块完成相对独立的功能，一个模块依赖于另一个模块。

javascript模块化开发的目的，一是`a piece of code` 包装成 `a useful unit`，二是`use other code` 升级成 `refer to other units`。

<!--more-->
---

## commonjs 

javascript的模块化开发，需要从commonjs说起。

javascript本来设计为在浏览器端环境执行的脚本语言，但强大的它怎么只甘心窝在浏览器端，它想要更开阔的天地，在java/python等语言的天空里留下它骚包的痕迹。但js在io流、文件系统、socket等方面的api是空白的，没有像python像java一样有标准库，所以CommonJS出现了，它要提供一个类似python标准库的东西，当然重心是在非浏览器应用环境下。

可怎么在自己代码里引入标准库还有各种三方库呢？

python、java等在服务器端应用的语言，在设计时候就有这样的机制，比如python有`import some_module`，java也有，它们有一套module加载或类加载的机制，而javascript没有，怎么办？CommonJS来了，它来定义一套这样的规范，nodejs对这套规范进行了部分实现。

首先用nodejs给个实际的效果:

{% highlight js %}
// a.js
console.log('a');

// b.js
console.log('b');

// c.js
var b=require('./b')
console.log('c')

// d.js
var c=require('./c'),
	a=require('./a')
console.log('d')

{% endhighlight %}

执行`node d.js` 对应的结果是：

	b
	c
	a
	d

如果学过python类似的是：

{% highlight python %}
// a.py
print 'a'

// b.py
print 'b'

// c.py
import b
print 'c'

// d.py
import c
import a
print 'd'
{% endhighlight %}

执行`python d.py` 结果是同样的，所以这个CommonJS规范符合预期的。

更典型点的module例子是：

{% highlight js %}
// cat.js
function Cat(){
	this.say = function(){
		console.log('miao~')
	}
}
module.exports = Cat

// main.js
var Cat = require('./cat')
var cat = new Cat()
cat.say()

// result is:
// miao~

{% endhighlight %}

如果是像python一样写javascript，这样不也还可以吗？它的缺陷在哪呢？

缺陷就在于，CommonJS规范是站在服务器端编程的立场来定义的。而javascript在浏览器端的运行环境是很不一样的，在服务器端的话，`require`的逻辑是定位文件、读取文件内容、执行返回，可浏览器端的环境下，依赖的脚本是需要通过http请求得到的，如果是先下载`main.js`，执行到`require`然后再发起网络请求下载`cat.js`再执行，这样的串行下载方式性能是不忍直视的。浏览器多个网络请求并发进行，这样的特性怎么能不利用呢?

---

## 动态 download script


由上所述，多个js请求是不能串行进行的。

多个请求并发请求不很简单吗，浏览器解析html中的请求时本就是这样子的啊，把所有依赖的script维护在html的head中，是本来通常的做法啊，这样有什么不好？

这又不得不回到模块化开发的起点和初衷了，模块化开发就是在系统变复杂以后才做的事，所以当很多script依赖关系复杂时候，这样就很不好。

比如不用模块化开发，javascript代码是这样写：
	
{% highlight javascript %}
(function () {
var $ = this.jQuery;

this.myExample = function () {};
}());
{% endhighlight %}

这样写代码的缺陷：

1. 例子代码中依赖的jQuery通过全局变量获得，当庞杂的依赖都放在全局作用域时，会产生混乱
2. 依赖very weakly stated，意思是开发者需要知道依赖的顺序，并在HTML中手动维护`<script>`的加载顺序，当系统庞大依赖关系复杂时候，这是个问题。

所以需要另外的方式按需动态加载依赖，并且要利用到浏览器并发请求的特性。

动态下载js的方法主要有：

1. XHR请求
	
	用XMLHttpRequest（XHR）请求到script。使用这种方法，会得到依赖的js代码，然后可以通过eval()或是新增script类型的节点，执行代码内容。

	缺陷一： XHR跨域请求有问题

	缺陷二： 难调试

	如果用eval()方法执行代码，还有其他缺陷，诸如[eval()干扰作用域链](http://www.nowamagic.net/librarys/veda/detail/1627)

2. Web worker

	javascript传统单线程模式，dom操作和后台IO在一个线程中运行相互牵制，而Web Workers 引入了多线程编程模式，一个 worker 是一个独立的线程，允许执行长期任务，同时也不影响页面响应。
	
	这个主要是个message-passing API， 用处主要是用户界面线程和后台服务线程之间信息交换，以防一些复杂的任务导致浏览器假死。
	
	缺陷主要是浏览器的支持不够广泛。seajs在非浏览器环境下通过`importScripts`加载依赖。


3. document.write()

	` document.write('<script type="text/javascript" src="jquery.js" > <\/script>');`

	可以用这样方式动态加载javascript代码，且与普通方式一样易于调试。

	缺陷一：A依赖B，在执行A代码前，B需先加载，而这个方法没法实现这点 `we do not have access to the script before it is executed`

	缺陷二：这个方式不能按需动态加载js

	缺陷三：这个方式加载js会阻塞页面渲染
	

4. head.appendChild(script)
	
	现在各种js模块加载器，RequireJS,seajs等都使用到第四种方案。
	
{% highlight javascript %}
var head = document.getElementsByTagName('head')[0],
script = document.createElement('script');

script.src = url;
head.appendChild(script);
{% endhighlight %}
	
好了，现在有了方法可以在js中按需下载相关依赖了。	
可是有了上面的第四种方法就够了吗？没有

浏览器的特性是，解析html时候遇到script请求，是下载后马上执行的。如果是第一种方法，是先下载下来脚本的内容，然后可以eval()执行，开发者是可以控制执行的时机的，否则的话按照CommonJS规范，直接把模块逻辑代码放在文件中，开发者是没法控制脚本的执行时机的，就没法保证依赖关系的前后顺序，怎么办呢?

---
## seajs

方法总是有的，如下用个函数包一层：

{% highlight js %}
define(function(require, exports, module) {

  // The business code goes here 业务逻辑代码在此

	
});
{% endhighlight %}

这样下载下来会立即执行`define函数`，基于`define`我们有机会掌控时机运行`business code`。这样一种模块定义方式，遵循CMD规范，由seajs进行实现。

具体可以看下seajs中对`define(id?,deps?,factory)`的实现，简单描述其逻辑是：

1. 依赖可以显式地声明在deps中，不然的话会自动解析
	
	如果在module code中有依赖，代码必定是包含`var dep1 = require('dep1')`之类的代码，所以可以根据factory函数的文本内容解析出相关依赖。

2. 生成模块实例，其中保存着模块元信息（meta），包括模块id，模块uri，模块的依赖deps，模块的业务逻辑代码factory等，并将实例放入缓存，实例状态为SAVED。

	模块实例在整个过程中有多个状态：
	  // 0 - START
	  // 1 - The `module.uri` is being fetched
	  FETCHING: 1,
	  // 2 - The meta data has been saved to cachedMods
	  SAVED: 2,
	  // 3 - The `module.dependencies` are being loaded
	  LOADING: 3,
	  // 4 - The module are ready to execute
	  LOADED: 4,
	  // 5 - The module is being executed
	  EXECUTING: 5,
	  // 6 - The `module.exports` is available
	  EXECUTED: 6,
	  // 7 - 404
	  ERROR: 7

除此之外并不做什么事。模块的定义部分到此完结了，看，本来在服务器端很简单平常的事情，在这里也费劲。

**可是依赖的脚本是什么时候下载的呢？**答案在`seajs.use(ids,callback)`,简单描述其逻辑是：

1. 每次调用`seajs.use`都会生成一个匿名module实例，它的uri按`data.cwd + "_use_" + cid()`自动生成，它的依赖由ids声明，callback绑定到此匿名实例
2. module实例load
	
	load module时先并行下载ids中声明的依赖，执行依赖文件中的`define`保存依赖模块元信息，然后对依赖进行相同的load操作，最终所有的直接依赖和间接依赖都load完毕

3. 执行依赖的module业务逻辑，通过exports暴露其中业务接口
	
	如果依赖的module业务逻辑还依赖另外的module，会调用`var dep=require('dep')`类似的代码，这会触发执行`间接依赖`的业务逻辑。

4. 执行callback

如果看代码，会觉得比较复杂，因为javascript的特性太灵活了，各种嵌套、回调，可了解下闭包的含义。用下面的例子简单说明：


{% highlight js %}
// a.html
<script> seajs.use("./main",function(){
	console.log('out main')
})</script>

// main.js
define(function(require, exports, module) {

   console.log('main')
   var hello = require('./hello')
   hello()
	
});

// hello.js
define(function(require, exports, module) {

   console.log('hello')
   return function(){
   		console.log('in hello function')
   }
});

// 结果控制台输出为:
// main
// hello 
// in hello function 
// out main

{% endhighlight %}

在输出`main`的调用栈为
![seajs-use-call-stack]({{ site.baseurl }}/assets/images/blog/seajs-use-call-stack.png)
在输出`hello`的调用栈为
![seajs-require-call-stack]({{ site.baseurl }}/assets/images/blog/seajs-require-call-stack.png)

基于seajs可以进行浏览器端的javascript模块化开发了。它有个特点是module的执行是在require调用里面触发的，所以执行的顺序是与代码的书写保持一致的。

这只是一种方式？与AMD有什么区别呢？

---

##AMD & CMD

例子：A依赖B。

基于B模块执行的时间点，分了两派，AMD和CMD。

它们都会保证`A模块使用到B模块代码`执行以前，B代码会先加载执行，不同点是AMD是`提前执行`，意思是`A模块代码执行`以前B模块就已经加载执行完毕，而CMD是`延迟执行`，意思是`A模块使用到B模块代码`以前B模块加载执行完毕。

AMD，也是种模块化开发js的规范，由RequireJS实现。AMD和CMD共同点：

1. 目的都是js的模块化开发
2. 在定义模块时声明依赖（显式或隐式）
3. 都是根据模块定义，动态加载依赖
4. 都是并行地发起js请求

不同点是：
1. 书写格式上
	
	书写格式AMD默认的是：

{% highlight js %}
define(
    //The name of this module
    "types/Manager",

    //The array of dependencies
    ["types/Employee"],

    //The function to execute when all dependencies have loaded. The
    //arguments to this function are the array of dependencies mentioned
    //above.
    function (Employee) {
        function Manager () {
            this.reports = [];
        }

        //This will now work
        Manager.prototype = new Employee();

        //return the Manager constructor function so it can be used by
        //other modules.
        return Manager;
    }
);
{% endhighlight %}

与CMD明显的区别是，AMD显式地声明依赖，当然AMD也可以用CMD的方式书写，自动解析依赖，但可能会引入误解.[为什么我不推荐使用 AMD 的 Simplified CommonJS wrapping](https://www.imququ.com/post/amd-simplified-commonjs-wrapping.html)。

2. 依赖中业务逻辑代码执行的时间点上

	我觉得，两者的区别主要是这个`时间点`的差别。用例子说明：

{% highlight html %}
//index-seajs.html
<html>
	<head>
		<script src="sea.js"></script>
	</head>
	<body>
		<script>
			console.log('seajs')
			seajs.use('./page')
		</script>
	</body>
</html>

//index-requirejs.html
<html>
	<head>
		<script src="require.js"></script>
	</head>
	<body>
		<script>
			console.log('RequireJS')
			require(['./page'])
		</script>
	</body>
</html>
{% endhighlight %}

{% highlight js %}
//page.js
define(function(require, exports, module) {
	var show_a_time=new Date();
   console.log(show_a_time,'show element a to user..') //模拟dom操作
   
   var subpage=require('./subpage') 
   var show_b_time=new Date()
   console.log(show_b_time,'show element b to user..')//模拟dom操作
   
   console.log('a,b显示间隔时间',(show_b_time-show_a_time)/1000)
});

//subpage.js
define(function(require, exports, module) {
   console.log(new Date(), 'task start.')
   sleep(3);// 模拟文件很大解析时间很长的一个情况
   console.log(new Date(), 'tast end.')

   function sleep(sec){
		var now = new Date(); 
		var exitTime = now.getTime() + sec*1000; 
		while (true) { 
			now = new Date(); 
			if (now.getTime() > exitTime) 
			return; 
		}
	} 
});

{% endhighlight %}

seajs结果如图: 
![seajs-render-interval]({{ site.baseurl }}/assets/images/blog/seajs-render-interval.png)

RequireJS结果如图:
![requirejs-render-interval]({{ site.baseurl }}/assets/images/blog/requirejs-render-interval.png)

当然这个例子意义不是很大。

对这个例子更形象点的说明是：它们区别就像上菜，RequireJS是先做好放着，上菜时候一个接着一个；seajs是做完一个就上一个，上菜间隔会长，但一开始不至于一直等着。你喜欢那个呢？







	





