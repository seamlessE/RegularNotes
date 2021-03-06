# 内存控制
基于无阻塞、事件驱动建立的Node服务，具有内存消耗低的优点，非常适合处理海量的网络请求。<br>
这一章算是正式迈进服务器编程的领域了，内存控制正式在海量请求和长时间运行的前提下进行探讨的。<br>
上一章节介绍了Node是如何利用CPU和I/O这两个服务器资源，本章介绍在Node中如何合理高效地使用内存。<br>
### V8的垃圾回收机制与内存限制
与java一样，由垃圾回收机制来进行自动内存管理，这使得开发者不需要像C/C++程序员那样在编写代码的过程中时刻关注内存的分配和释放问题。<br>
其实，对于性能敏感的服务端程序，内存管理的好坏、垃圾回收状况是否优良，都会对服务构成影响。<br>
**而在Node中，这一切都与Node的JavaScript执行引擎V8息息相关。**<br>
#### V8的内存限制
在Node中通过JavaScript使用内存时就会发现只能使用部分内存（64位系统约1.4GB，32位系统约0.7GB）。在这样的限制下，将会导致Node无法直接操作大内存对象，比如无法将一个2GB的文件读入内存中进行字符串分析处理，即使物理内存有32GB。这样，在单个Node进程的情况下，计算机的内存资源无法得到充足的使用。<br>
问题的主要原因：Node基于V8构建，所以Node中使用的JavaScript对象基本上都是通过V8自己的方式来进行分配和管理的。
#### V8的对象分配
在V8中，所有的JavaScript对象都是 **通过堆来进行分配的**。Node提供了V8中内存使用量的查看方式
```js
node
> process.memoryUsage();
{ rss: 21725184,
  heapTotal: 7684096,
  heapUsed: 4949968,
  external: 8752 }
```
memoryUsage()返回3个属性，heapTotal和heapUsed是V8的堆内存使用情况，前者是已申请到的堆内存，后者是当前使用的量。<br>
![heapStatus](./img/node_heap_status.png "V8堆内存机制示意图")<br>
当我们在代码中声明变量并赋值时，所使用对象的内存就分配在堆中。如果已申请的堆空闲内存不够分配新的对象，将继续申请堆内存，直到堆的大小超过V8的限制为止。<br>
至于V8为何要限制堆的大小，原因两个<br>
表层：V8最初为浏览器而设计，不太可能遇到用大量内存的场景，对于网页，V8限制值已经绰绰有余。<br>
深层：V8的垃圾回收机制的限制。官方说法，以1.5GB的垃圾回收堆内存为例，V8做一次小的垃圾回收需要50ms以上，做一次非增量式的垃圾回收甚至需要1秒以上。这是垃圾回收中引起js线程暂停执行的时间，在这样的时间花销下，应用的性能和响应能力都会直线下降。因此，直接限制堆内存是一个好选择。<br>
当然限制也可以打开，`node --max-old-space-size=1700 xx.js`或`node --max-new-space-size=1024 xx.js` 更改内存大小<br>
#### V8的垃圾回收机制
介绍下V8用到的各种垃圾回收算法。<br>
1. V8 主要的垃圾回收算法
V8的垃圾回收策略主要基于 **分代式垃圾回收机制**。在自动垃圾回收的演变过程，人们发现没有一种垃圾回收算法能够胜任所有的场景。<br>
因为在实际应用中，对象的生存周期长短不一，不同的算法只能针对特定情况具有最好的效果。为此，统计学在垃圾回收算法的发展中产生了较大的作用，现代的垃圾回收算法中按对象的存活时间将内存的垃圾回收进行不同的分代，然后分别对不同分代施以更高效的算法。<br>
* V8的内存分代
新生代和老生代。新生代中的对象为存活时间较短的对象，老生代中的对象为存活时间较长或常驻内存的对象。<br>
|新生代的内存空间|☁️ ☁️☁️☁️☁️☁️☁️☁️☁️☁️老生代的内存空间☁️☁️☁️☁️☁️☁️☁️☁️☁️☁️☁️☁️|<br>
V8堆的整体大小就是新生代所用内存空间加上老生代的内存空间。--max-old-space-size可设置老生代内存空间的最大值，--max-new-space-size设置新生代的内存空间大小。<br>
以上两个命令需要在启动时就指定。这意味着V8使用的内存没有办法根据使用情况自动扩充，当内存分配过程中超过极限值时，就会引起进程出错。<br>
对于新生代内存，它由两个`reserved_semispace_size_`所构成。按机器位数不同，`reserved_semispace_size_`在64位系统和32位系统上分别16MB和8MB。所以新生代内存的最大值在64位系统和32位系统上分别为32MB和16MB。<br>
V8堆内存的最大保留空间可以从下面的代码中看出来，其公式为4*reverved_semispace_size_+max_old_generation_size_:
* Scavenge算法
在分代的基础上，新生代中的对象主要通过Scavenge算法进行垃圾回收。在Scavenge的具体实现中，主要采用了Cheney算法。<br>
Cheney算法是一种采用复制的方式实现的垃圾回收算法。它将堆内存一分二，每一部分空间称为semispace。在这两个semispace空间中，只有一个处于使用中，另一个处于闲置状态。处于使用状态的semispace空间称为From空间，处于闲置状态的空间称为To空间。当我们分配对象时，先是在From空间中进行分配。当开始进行垃圾回收时，会检查From空间中的存活对象，这些存活对象将被复制到To空间中，而非存活对象占用的空间将会被释放。完成复制后，From空间和To空间的角色发生对换。简而言之，在垃圾回收的过程中，就是通过将存活对象在两个semispace空间之间进行复制。<br>
Scavenge的缺点是只能使用堆内存的一半，这是由划分空间和复制机制所决定的。但Scavenge由于只复制存活的对象，并且对于生命周期短的场景存活对象只占少部分，所以它在时间效率上有优异的表现。<br>
所以Scavenge是典型的牺牲空间换取时间的算法，所以无法大规模地应用到所有的垃圾回收中。但可以发现，Scavenge非常适合应用在新生代中，因为新生代中对象的生命周期较短，恰恰适合这个算法。<br>
所以V8的堆内存示意图<br>
![V8堆内存示意图](./img/v8_memory.png "V8堆内存示意图")<br>
实际使用的堆内存是新生代中的两个semispace空间大小和老生代所用内存大小之和。<br>
当一个对象经过多次复制依然存活时，它将会被认为是生命周期较长的对象。这种较长的周期的对象将会被移动到老生代中，采用新的算法进行管理。**对象从新生代中移动到老生代中的过程称为晋升。**<br>
对象的晋升的条件主要有两个，一个是对象是否经历过Scavenge回收，一个是To空间的内存占用比超过限制。<br>
晋升流程<br>
![晋升流程](./img/jinshengscaven.png)<br>
to空间内存占用比<br>
![晋升判断示意图](./img/jinsheng25.png)<br>
如果占比过高，会影响后续的内存分配。<br>
* V8在老生代中主要采用Mark-Sweep & Mark-Compact相结合的方式进行辣鸡回收
对于老生代中的对象，由于存活对象占较大比重，再采用Scavenge的方式会有两个问题：
> 一个是存活对象较多，复制存活对象的效率将会很低。
> 另一个问题依然是浪费一半的空间的问题
Mark-Sweep是标记清楚的意思，它分为标记和清除两个阶段。与Scavenge相比，Mark-Sweep并不将内存空间划分为两半，所以不存在浪费一般空间的行为。<br>
与Scavenge复制活着的对象不同，Mark-Sweep在标记阶段遍历堆中所有对象，并标记活着的对象，在随后的清除阶段中，只清除没有被标记的对象。<br>
可以看出，Scavenge只复制活着的对象，而Mark-sweep只清理死亡对象。活对象在新生代中只占小部分，死亡对象在老生代中只占小部分，这就是两种回收方式能高效处理的原因。<br>
Mark-Sweep清除后，内存空间会出现不连续的状态。这种内存碎片会对后续的内存分配造成问题，因为很可能出现需要分配一个大对象的情况，这时所有的碎片空间都无法完成此次分配，就会提前触发辣鸡回收，而这次回收是不必要的。<br>
所以为了解决上面所说的问题，提出了Mark-Compact。<br>
* Mark-Compact是标记整理的意思，是在Mark-Sweep的基础上演变而来的。它们的差别在于对象在标记死亡后，在整理的过程中，将或者的对象往一端移动，移动完成后，直接清理掉边界外的内存。完成移动后，就可以直接清除最右边的存活对象后面的内存区域完成回收。<br>
* Incremental Marking
垃圾回收会将应用逻辑暂停下来，待执行完垃圾回收后再恢复执行引用逻辑，这种行为被称为“全停顿”。为了降低全堆垃圾回收带来的停顿时间，V8从标记阶段入手，将原本要一口气停顿完成的动作改为 **增量标记（incremental Marking）**。<br>
辣鸡回收与应用逻辑交替执行直到标记阶段完成。<br>
V8后续还引入了延迟清理与增量式整理（incremental compaction），让清理与整理动作也变成增量式的。同时还计划引入并行标记与并行清理，进一步利用多核性能降低每次停顿的时间。<br>
##### 2.小结
从V8的自动辣鸡回收机制的设计角度看到，V8对内存使用进行限制的缘由。新生代设计为一个较小的内存空间是合理的，而老生代空间过大对于辣鸡回收并无特别意义。V8对内存限制的设置对于Chrome浏览器这种每个选项卡页面使用一个V8实例而言，内存的使用是绰绰有余的。对于Node编写的服务器端来说，内存限制也并不影响正常场景下的使用。但是对于V8的辣鸡回收特点和JavaScript在单线程上的执行情况，辣鸡回收是影响性能的因素之一。想要高性能的执行效率，需要注意让辣鸡回收尽量少地进行，尤其是全堆垃圾回收。<br>
### 高效使用内存
在V8面前，开发者所要具备的责任是如何让辣鸡回收机制更高效地工作。<br>
#### 作用域
提到如何出发辣鸡回收，第一个要介绍的是作用域（scope）。在JS中能行程作用域的有函数调用、with以及全局作用域。<br>
```js
var foo = function(){
  var local = {};
};
```
在这个示例中，由于对象非常小，将会分配在新生代中的From空间中。在作用域释放后，局部变量local失效，其引用的对象将会在下次辣鸡回收时被释放。<br>
##### 1.标识符查找
与作用域相关的即是标识符查找。所谓标识符，可以理解为变量名。在下面的代码中，执行bar()函数时，将会遇到local变量。 （这里解释的作用域一般是冒泡查找）
```js
var bar = function(){
  console.log(local);
};
```
##### 2.作用域链
```js
var foo = function(){
  var local = 'local var';
  var bar = function(){
    var local = "another var";
    var baz = function(){
      console.log(local);
    };
    baz();
  };
  bar();
};
foo();
```
输出 another var<br>
##### 3.变量的主动释放
如果变量是全局变量（不通过var声明或定义在global变量上），由于全局作用域需要直到进程退出才能释放，此时将导致引用的对象常驻内存（常驻在老生代中）。如果需要释放常驻内存的对象，可以通过delete操作来删除引用关系。或者将变量重新赋值，让旧的对象脱离引用关系。在接下来的老生代内存清除和整理的过程中，会被回收释放。
```js
global.foo = "i am global object";
console.log(global.foo);
delete global.foo;
global.foo = undefined;
console.log(global.foo);
```
如果在非全局作用域中，想主动释放变量引用的对象，也可以通过这样的方式。虽然delete操作和重新赋值具有相同的效果，但是V8中通过delete删除对象的属性有可能干扰V8的优化，所以赋值方式解除引用会更好。<br>
#### 闭包?
在js中，实现外部作用域访问内部作用域中变量的方法叫做闭包（closure）。这得益于高阶函数的特性：函数可以作为参数或者返回值
```js
var foo = function(){
  var bar = function(){
    var local = "insert";
    return function(){
      return local;
    };
  };
  var baz = bar();
  console.log(baz());
};
```
#### 小结
在正常的js执行中，无法立即回收的内存有闭包和全局变量引用这两种情况。由于V8的内存限制，要十分小心此类变量是否无限制地增加，因为他会导致老生代中的对象增多。<br>
### 内存指标
一般而言，应用中存在一些全局性的对象是正常的，而且在正常使用中，变量都会自动释放回收。但是也会存在一些我们认为会回收但是却没有被回收的对象，这会导致内存占用无限增长。一旦增长达到V8的内存限制，将会得到内存溢出错误，进而导致进程推出。<br>
#### 查看内存使用情况
`process.memoryUsage()`;<br>
```js
bogon:~ mrtrans$ node
> process.memoryUsage
[Function: memoryUsage]
> process.memoryUsage()
{ rss: 21843968,
  heapTotal: 7684096,
  heapUsed: 5055272,
  external: 8782 }
> 
```
rss是resident set size的缩写，即进程的常驻内存部分。进程的内存总共有几部分，一部分是rss，其余部分在交换区（swap）或者文件系统（filesystem）。<br>
除了rss外，heapTotal和heapUsed对应的是V8的堆内存信息。heapTotal是堆中总共申请的内存量，heapUsed表示目前堆中使用中的内存量。这三个值的单位都是字节。<br>
```js
var showMem = function(){
  var mem = process.memoryUsage();
  var format = function(byytes){
    ...
  }
  ...
}
```
##### 查看系统的内存占用
```js
> os.totalmem()
8589934592
> os.freemem()
331943936
```
#### 堆外内存
堆中的内存用量总是小于进程的常驻内存用量，这意味着Node中的内存使用并非都是通过V8进行分配的。我们将那些不是通过V8分配的内存称为堆外内存。<br>
实现
```js
var useMem = function(){
  var size = 200*1024*1024;
  var buffer = new Buffer(size);
  for(var i = 0;i<size;i++){
    buffer[i] = 0;
  }
  return buffer;
};
```
#### 小结
Node的内存构成主要由通过V8进行分配的部分和Node自行分配的部分。受V8的垃圾回收限制的主要是V8的堆内存。
### 内存泄漏
Node对内存泄漏十分敏感，一旦线上应用有成千上万的流量，哪怕是一个字节的内存泄漏也会造成堆积，辣鸡回收过程中将会耗费更多时间进行对象扫描，应用响应缓慢，直到进程内存溢出，应用崩溃。<br>
内存泄漏：应当回收的对象出现意外而没有被回收，变成了常驻在老生代中的对象。<br>
* 缓存
* 队列消费不及时
* 作用域未释放
#### 慎将内存当作缓存
一旦一个对象被当做缓存来使用，那就意味着它将会常驻在老生代中。缓存中存储的键越多，长期存活的对象也就越多，这将导致辣鸡回收在进行扫描和整理时，对这些对象做无用功。<br>
js开发者通常喜欢用对象的键值对来缓存东西，但这与严格意义上的缓存又有着区别，严格意义的缓存有着完善的过期策略，而普通对象的键值对并没有。<br>
如下代码所示：
```js
var cache = {};
var get = function(key){
    if(cache[key]){
        return cache[key];
    }else{
    }
};
var set = function(key,value){
    cache[key] = value;
};
```
这个利用了js对象创建一个缓存对象，但是受辣鸡回收机制的影响。只能小量使用。全局变量太多！！！内存无限制增长！所以在Node中，任何试图拿内存当缓存的行为都应当被限制。当然，这种限制并不是不允许你使用，就是要你小心。
##### 缓存限制策略
为了解决缓存中的对象永远无法释放的问题，需要加入一种策略来限制缓存的无限增长。
```js
var limitableMap = function(limit){
  this.limit = limit || 10;
  this.map = {}; // 缓存
  this.keys = []; // 内存
};
var hasOwnProperty = Object.prototype.hasOwnProperty;
limitableMap.prototype.set = function(key,value){
  var map = this.map;
  var keys = this.keys;
  if(!hasOwnProperty.call(map,key)){
    if(keys.length === this.limit){
      var firstkey = keys.shift();
      delete map[firstkey];
    }
    keys.push(key);
  }
  map[key] = value;
};
LimitableMap.prototype.get = function(key){
  return this.map[key];
};
module.exports = LimitableMap;
```
记录键在数组中，一旦超过数量，就以先进显出的方式进行淘汰。<br>
模块机制，为了加速模块的引入，所有模块都会通过编译执行，然后被缓存起来。由于通过exprots导出的函数，可以访问文件模块中的私有变量，这样每个文件模块在编译执行后形成的作用域因为模块缓存的原因，不会被释放。<br>
举个🌰
```js
(function(exports,require,module,__filename,__dirname){
  var local = "letvarible";
  exports.get = function(){
    return local;
  };
});
```
由于模块的缓存机制，模块是常驻老生代的。在设计模块时，要十分注意内存泄漏的出现。<br>
举个🌰
```js
var lekArray = [];
exports.leak = function(){
  leakArray.push("leak"+Math.random());
};
```
这里每次调用leak()方法时，都导致局部变量leakArray不停增加内存的占用，且不被释放。如果模块要这么设计，请添加清空队列的相应接口，以供调用者释放内存。<br>
##### 缓存的解决方案
直接将内存作为缓存的方案要十分慎重。除了限制缓存的大小外，另外要考虑的事情是，进程之间是无法共享内存。如果在进程内使用缓存，这些缓存不可避免地有重复，对物理内存的使用是一种浪费。<br>
采用进程外的缓存，是解决使用大量缓存的方案，进程自身不存储状态。<br>
**外部的缓存软件**<br>
* 将缓存转移到外部，减少常驻内存的对象的数量，让垃圾回收更高效。
* 进程之间可以共享缓存。
#### 关注队列状态
另一个不经意产生的内存泄漏是队列。在js中可以通过队列（数组对象）来完成许多特殊的需求，比如Bagpipe。队列在消费者-生产者模型中经常充当中间产物。这是一个容易忽略的情况，因为大多数应用场景下，消费的速度远远大于生产速度，内存泄漏不易产生。但是一旦反过来，低于了，就会形成堆积。<br>
* 深度解决方案是监控队列的长度，一旦堆积，应当通过监控系统产生警报并通知相关人员。<br>
* 另一种解决方案是任意异步调用都应该包含超时机制，一旦在限定的时间内未完成相应，通过毁掉函数传递超时异常，使得任意异步调用的回调都具备可控的相应时间，给消费速度一个下限值。<br>
* 两种模式，超时和拒绝模式。两种模式都有效防止队列拥塞导致内存泄漏的问题<br>
### 内存泄漏排查
在Node中，由于V8的堆内存大小的限制，它对内存泄漏非常敏感。下面有几个排查方案<br>
排查内存泄漏的原因主要通过对堆内存进行分析而找到<br>
🙆 v8-profiler<br>
🙆‍♂️ node-heapdump<br>
🙆‍♂️ node-mtrace <br>
🙆 dtrace<br>
🙆‍♂️ node-memwatch<br>
### 大内存应用
在Node中，不可避免地还是会存在操作大文件的场景。由于node的内存限制，操作大文件也需要小心，好在Node提供了stream模块用于处理大文件。<br>
stream模块是Node的原生模块，直接引用即可<br>
stream继承EventEmitter，具备基本的自定义事件功能，同时抽象出标准的事件和方法。它分为可读和可写两种。Node中的大多数模块都有stream的应用。🌰 fs的createReadStream()和createWriteStream()方法可以分别用于创建文件的可读流和可写流，process stdin和stdout<br>
通过fs的createReadStream()和createWriteStream()方法来进行大文件的操作
```js
var reader = fs.createReadStream('in.txt');
var writer = fs.createWriteStream('out.txt');
reader.on('data',function(chunk){
  writer.write(chunk);
});
reader.on('end',function(){
  writer.end();
});
```
```js
var reader = fs.createReadStream('in.txt');
var writer = fs.createWriterStream('out.txt');
reader.pipe(writer);
```
可读流提供了管道方法pipe()，封装了data事件和写入操作。
### 最后
Node将js的主要饮用场景扩展到服务器端，相应要考虑的细节也与浏览器不同，需要更严谨地为每一份资源作出安排。总的来说，内存在Node中不能随心所欲地使用，但也不是完全不擅长。
