# Generators
回顾下第二节提到的使用 callback 进行异步编程的弊端：
- 不符合我们大脑按步骤进行的工作模式

- 因为 IoC 的问题，callback 并不安全可信

在第三节中出现的 Promise 很好的解决了👆安全可信的问题，但对于符合大脑的工作模式，依然无能为力。而接下去我们的探索按照同步模式实现的异步编程 —— Generator 异步编程的流程控制，就是解决这个问题的一剂良药。

## 打破 “一跑就到底”(Breaking Run-to-Completion)
众所周知，在 JS 中，普通的函数一旦开始执行，就只能执行到底，没有机制能够打断它的执行。

而 ES6 引入的 generator 是一个新类型的函数，它能够打破这个 “无法停机” 问题。

一个简单的思想实验帮助我们更好的理解：

```js
var x = 1;

function foo () {
  x++;
  bar();
  console.log('x: ', x);
}

function bar () {
  x++;
}

foo(); // x: 3
```

👆显而易见 `x` 的结果是 `3`；但若当 `bar()` 没有在 `foo()` 中被调用，而又想让 `x` 的结果不变，有什么办法能实现呢？

在 可抢占式的多线程语言(preemptive multithreaded languages) 中，当某个线程执行到 `x++` 时，另一个线程就能见缝插针的调用 `bar()`，从而将结果维持不变。但 JS 不仅不能实现 “抢占”，而且它还是单线程的，那这样的实现也就无从谈起了。

不过，有了 generator 之后，实现这样的 协同并发(cooperative concurrency) 机制成为了可能，因为你能暂停 `foo()` 的执行：

```js
var x = 1;

function *foo () {
  x++;
  yield;
  console.log('x: ', x);
}

function bar () {
  x++;
}

var it = foo();
it.next();
x; // 2
bar();
x; // 3
it.next(); // x: 3
```

**Note**：关于 `*` 的位置，牵扯到了风格的选择，比如有的文档或代码中是将 `*` 靠近 `function`：`function* foo () {……}`。而作者选择将 `*` 靠近函数名是为了更便于表达，就比如 `*foo()` 和 `foo()` 的区别一望便知。

上面整个 generator 函数 `*foo()` 执行的过程如下：
  1. `it = foo()` 并不是执行 generator，而是相当于构造了一个 *iterator* 对象，用来控制整个 generator 的执行；

  2. 第一次调用 `it.next()` 才是正式开始执行 generator，并且执行了第一行 `x++` 代码；

  3. 随后遇到了第一个 `yield` 关键字，函数 `*foo()` 的执行就暂停了；

  4. 而后我们输入 `x` 得到了其当前的值为 `2`；

  5. 接下去执行了 `bar()`，在其内部执行了 `x++` 的代码；

  6. 再一次检查 `x` 的值为 `3`； 

  7. 最终，调用 `it.next()` 将暂停的 generator 函数恢复执行，并控制台打印出了 `"x: 3"` 的信息……

总的来讲，在 generator 函数中，通过关键字 `yield` 就能暂停 generator 函数的执行，而后可以通过调用 `next()` 恢复函数的执行。并且，函数执不执行完显然不是强制要求的 —— 看上去没啥了不起？

### 输入和输出(Input and Output)
generator 函数本质上来看依然是函数，因此一些普通函数的基本特性依然对它有效，就比如 “输入” —— 参数 和 “输出” —— 返回值：

```js
function *foo(x, y) {
  return x * y;
}

var it = foo();

var res = it.next(6, 7);

res.value; // 42
```

👆 我们将 `6` 和 `7` 作为 `*foo(…)` 的参数传入，而后计算出 `6 * 7` 的值后返回 —— 除了在激活 generator 函数的形式上，这些操作好像和普通的函数调用没有什么太大的区别。但细节却不能忽视：`*foo(6, 7)` 并不是函数调用，而是构建出了一个 iterator 对象；`it.next()` 才开始执行 generator 的函数体内容，而其返回值是一个包含了 `value` 属性的对象；而 `value` 的值则是 `yield` 或 `return` 输出的值。

但为什么我们需要一个可迭代的对象来控制 generator 函数呢？让它自执行不好吗？

#### 迭代的信息(Iteration Messaging)
除了用参数和返回值实现 “输入” 和 “输出” 之外，通过关键字 `yield` 和 迭代对象的 `next(…)` 方法，能够实现颗粒度更细，更有用的 “输入” 和 “输出”：

```js
function *foo(x) {
  const baseInd = Math.round(Math.random() * 10);
  var y = x * (yield baseInd);
  return y;
}

var it = foo(6);

var res = it.next();

res.value;

res = it.next(res.value * 5);

res.value;
```

`foo(6)` 将 `6` 作为参数传入 generator 函数，调用 `it.next()` 后，获得 `(yield baseInd)` 中的 `baseInd`，而后将其乘以 `5`，并作为参数再次传递到 generator 中，最终生成的值是这一系列参数传递的组合，即 `6 * baseInd * 5`。

需要强调的是，在上面执行 generator 的过程中，有一个有趣的现象 —— `next(…)` 始终比 `yield` 多一个。有些人误以为这是个错配，但导致这个 “错配” 的原因是因为第一个 `next(…)` 的作用是启动 generator 函数，而后在遇到第一个 `yield` 关键字后暂停整个函数的后续执行 —— 这即意味着说，你想要获得 `yield` 之后的结果，至少还需要执行一次 `next(…)` 才行。

##### 两个问题的故事(Tale of Two Question)
换个角度来看这个所谓的 “错配” 的问题可能会得到不一样的解读。

类似于服务端和客户端通过API进行通信一样，我们将 generator 的内部和外部的 iterator 同样视为是服务端和客户端：

```js
function *server(x) {
  const baseInd = Math.round(Math.random() * 10);
  var y = x * (yield baseInd);
  return y;
}

var client = server(6); // 和服务端建立了连接，并发送了 6 作为参数

var res = client.next(); // 客户端：服务端，请把第一个返回值告诉我，谢谢

res.value; // 一个由服务端生成的值

res = client.next(res.value * 5); // 将这个值乘以 5 发送给服务端

res.value; // 得到服务端最终生成的值
```

👆 一个类似于 “半双工通信” 的机制由此而来，并且由于第一次调用 `next()` 的时候，本质上来看就是启动 generator 函数，若是将它想象成函数一开始就有一个隐藏的 `yield`，并且从这里出发，那么不用传递任何参数(argument-free)也是在情理之中的。

最后的 `return y` 标示着整个 generator 函数执行完毕。若是没有显示的写 `return` 的代码，那么和普通函数一样，JS 引擎会隐式的帮你加上 `return undefined;` 的语句。
 
这种通过 `yield` 和 `next(…)` 实现的通信机制非常强大，但离异步流程控制好像还有点距离？！别急，接着看下去。

### 多个迭代器(Multiple Iterators)
每次调用 genenrator 函数，都会创建一个新的 iterator 实例，即意味着你能同时创建多个 iterator 的实例，而且它们还能相互作用：

```js
function *foo() {
  var x = yield 2;
  z++;
  var y = yield (x * z);
  console.log(x, y, z);
}

var z = 1;

var it1 = foo();
var it2 = foo();

var val1 = it1.next().value; // 2
var val2 = it2.next().value; // 2

val1 = it1.next(val2 * 10).value; // 40
val2 = it2.next(val1 * 5).value; // 600

it1.next(val2 / 2); // 2 300 3
it2.next(val1 / 4); // 200 10 3

```

👆上面的逻辑有点绕，而且也没人会在真实的开发中这么做，但并不妨碍我们将它作为一个案例来研究多个 iterator 的实例能够实现互相影响的现象。

简单分析下过程：
1. `it1` 和 `it2` 是两个由 `*foo` 创建的 iterator 实例，它们第一次调用的 `next()` 方法并获取返回值中的 `value` 属性时，其结果都是 `2`；

2. `val2 * 10` 就相当于 `2 * 10`，其结果 `20` 作为参数传递给 `it1`，并将其赋值给变量 `x`，而后变量 `z` 自增的结果是 `2`，因此 `x * z` 的结果相当于 `20 * 2` 即 `40`，并将其作为返回值复制给 `val1`；

3. `val1 * 5` 相当于 `40 * 5` 即 `200`，并作为参数传入 `it2` 中，此时 `it2` 会经历 `it1` 的步骤，只是 `x` 和 `z` 数值发生了变化，分别是 `200` 和 `3`，因此最终 `val2` 的值变成了 `600`(来自 `yield (x * z)` 的返回值)；

4. `val2 / 2` 的结果 `300` 作为参数再一次传递到 `it1` 中，而后赋值给了 `y`，因此 `it1` 最终打印的结果是 `2 300 3`；

5. 同理可得 `val1 / 4` 的结果为 `40 / 4`，而后传入 `it2`，打印的结果为 `200 10 3`


#### 交错进行(interleaving)
回顾下之前提到的普通函数的运行模式：

```js
var a = 1;
var b = 2;

function foo () {
  a++;
  b = b * a;
  a = b + 3;
}

function bar () {
  b--;
  a = 8 + b;
  b = a * 2;
}
```

👆上面的代码无论是 `foo` 或 `bar` 先执行，最终都只能得到两种不一样的结果。但是，若我们使用 generator 函数改造一下，那就能得到很多不一样的结果：

```js
var a = 1;
var b = 2;

function *foo() {
  a++;
  yield;
  b = b * a;
  a = (yield b) + 3;
}

function *bar() {
  b--;
  yield;
  a = (yield 8) * b;
  b = a * (yield 2);
}
```

👆 `*foo()` 和 `*bar()` 不同的调用顺序会产生不一样的结果，而用这个特性来模拟 “线程竞争(threaded race conditions)” 也是可行的 —— 比如这个函数有副作用，依赖全局变量啥的：

先实现一个工具函数用来简化 iterator 的执行：

```js
function step (gen) {
  var it = gen();
  var last;

  return function () {
    last = it.next(last).value;
  }
}
```

接下来，仅仅是为了实验的目的，我们再实现因多个 iterator 交互而相互影响的例子：

```js
a = 1;
b = 2;

var s1 = step(foo);
var s2 = step(bar);

s1();
s1();
s1();

s2();
s2();
s2();
s2();

console.log(a, b); // 11 22
```

若是你真的理解了 generator 函数的执行机制，那得出结果并不困难。若是调换一下 `s1()` 和 `s2()` 的调用顺序的话，结果会又不一样：

```js
// ……
s1();
s2();
s2();
s1();
s2();
s1();
s2();
// ……
```

👆 通过 generator 改造，其最终可能的结果的个数，和普通函数只有两个结果相比起来大大增加。不过，通常我们并不会这么写代码，但这个 “并发” 的能力其实有大用处，后面细说。

## Generator生成的值(Generator'ing Values)
之前的一些尝试只是开胃菜，要搞清楚 generator 的原理，依然需要一步步的分析实现的细节，譬如我们先从 iterator 入手。

### 制造者和迭代器(Producers and Iterators)
想象一下有这个场景：你需要生成多个互相关联的值，当前值的状态依赖之前的值。为了实现这个需求，一般你需要一个状态管理器，来帮忙记住每一个之前的值的状态。

比如，可以借助闭包来实现：

```js
var giveMeSomething = (function () {
  var nextVal;

  return function () {
    if (nextVal === undefined) {
      nextVal = 1;
    } else {
      nextVal = (3 * nextVal) + 6;
    }

    return nextVal;
  };
})();

giveMeSomething(); // 1
giveMeSomething(); // 9
giveMeSomething(); // 33
giveMeSomething(); // 105
```

首先，这种方式👆的实现会有内存泄漏的问题；其次，若是用这种方式来 “记住” 的非简单类型的值，那用起来就很繁琐了。

实际上，对于上面的这种情况，在 JS 的实现中，已经有很通用的解决方案了：和大多数的语言一致，在 JS 中，实现一套调用 `next()` 方法，来获得返回的值的机制，即可满足调用状态的保留：

```js
var something = (function () {
  var nextVal;

  return {
    [Symbol.iterator]: function () {
      return this;
    },
    next: function () {
      if (nextVal === undefined) {
        nextVal = 1;
      } else {
        nextVal = (3 * nextVal) + 6;
      }

      return { done: false, value: nextVal };
    }
  }
})();

something.next().value; // 1
something.next().value; // 9
something.next().value; // 33
something.next().value; // 105
```

调用 `next()` 而得到的返回值是一个包含了两个属性的对象：`done` 和 `value` 分别代表是迭代完成的状态和本次迭代的值。

ES6 新增的 `for…of` 循环是一个标准的可自动迭代的循环语法：

```js
for (const v of something) {
  console.info(v);
  if (v > 500) {
    break;
  }
}
// 1 9 33 105 321 969
```

因为 `something` 迭代器返回的 `done` 始终为 `false`，因此只能手动在条件满足后，`break` 终止循环，否则就会一直循环下去。

`for…of` 循环在每次迭代的过程中，都会调用被循环对象的 `next()` 方法，但不会传递任何参数进去。当 `done` 为 `true` 的时候，自动结束循环迭代 —— 这整个机制十分适合用来遍历一组数据。

当然，你也能手动使用普通的 `for` 循环，来模拟这个过程：

```js
for (let ret = something.next(); !ret.done; ret = something.next()) {
  console.info(ret.value);
  if (ret.value > 500) {
    break;
  }
}
```

👆虽然没有 `for…of` 循环来的优雅，但好处是能够往 `next(…)` 中传递需要的值。

许多 JS 原生的值，都内置了一个默认的迭代器，比如数组(列表)：

```js
var a = [1, 3, 5, 7, 9];

for (const val of a) {
  console.info(val);
}
// 1 3 5 7 9
```

**Note**：ES6 中并没有为普通的 `object` 内置迭代器，其中一个缘由是没办法保证迭代的顺序，除此之外，若是普通对象内置了迭代器，那么其他继承自普通对象的其他数据类型都会受到影响。但有一个折中的办法能让普通的对象也能用上 `for…of` —— 调用 `Object.keys(…)` API，获取基于普通对象属性名的数组，而后在遍历它，并挨个取出属性值。

### 可迭代对象(Iterable)
`iterable` 可迭代对象就是一个包含了迭代器的 js 对象，比如上面提到的 `something` —— 它实现了 `next()` 方法以及定义了 `Symbol.iterator` 接口。

在 ES6 中，中可迭代对象中获取迭代器只能通过调用 `Symbol.iterator` 所定义的函数，当这个函数被调用时，应该返回一个迭代器。并且，虽然不是强制性的，但每次调用都应该返回一个全新的迭代器。

比如，在上例中的数组 `a`，当把它放到 `for…of` 循环中时，会自动调用它内置的 `Symbol.iterator`。既然如此，我们当然也能手动的调用它：

```js
var a = [1, 3, 5, 7, 9];

var it = a[Symbol.iterator]();

it.next().value; // 1
it.next().value; // 3
it.next().value; // 5
```

不知道你有没有注意到之前在定义 `something` 的时候，它的 `Symbol.iterator` 是这样实现的：

```js
[Symbol.iterator]: function () { return this; }
```

而后若是将它用在 `for…of` 中的话：

```js
var a = [1, 3, 5, 7, 9];

for (const val of something) {
  console.info(val);
  if (val > 500) break;
}
// 1 3 5 7 9
```

👆虽然看上去有点怪，但这行得通，因为当 `for…of` 调用 `something` 的时候，它确定义了 `Symbol.iterator`，并且返回了它自己 `this`，而它自己定义了 `next()` 方法，符合规范，于是接下去就能进行循环遍历了。

### 生成器中的迭代器(Generator Iterator)
生成器(generator) 从其功能上看，可以将其视为是一个生成迭代器的函数，该函数上定义了 `next()` 方法。

因此，从技术角度来看，generator 并非是一个可迭代的对象，尽管它们好像差不多，但当你调用 generator 的时候，它的返回值是一个迭代器：

```js
function *foo() {};

var it = foo(); // it 是迭代对象
```

当然，我们同样能够用 generator 来实现一个 `something`，用来执行一个无限循环的迭代：

```js
function *something() {
  let nextVal;

  while (true) {
    if (nextVal === undefined) {
      nextVal = 1;
    } else {
      nextVal = (3 * nextVal) + 1;
    }

    yield nextVal;
  }
}
```

**Note**：虽然非常不推荐，在真实的代码中使用不带任何 `break` 或 `return` 语句的 `while(true) {…}` —— 因为这样会造成浏览器的UI线程卡死；但在 generator 中，并没有这个问题，因为 generator 中每一次的 `yield` 会暂停 `while(true) {…}`，等到 JS主线程 的代码都执行完毕，才会再次执行。

由于每次 `yield` 都会保留 `*something()` 的调用栈，因此也用不着用一套很八股的模板，配合闭包(closure)，来完成对当前调用栈的状态记录 —— 这可不是仅仅简化了代码而已 —— 不仅不用自己手动定义 iterator 接口，而且代码也更语义化，能够清楚的表达它的职责所在。比如，`while(true) {…}` 循环就是告诉我们 generator 可以无限执行下去。

把 `*something()` 和 `for…of` 循环组合起来看看和之前的区别：

```js
for (const v of something()) {
  console.info(v);
  if (v > 500) {
    break;
  }
}
// 1 9 33 105 321 969
```

倘若是你仔细观察👆上面的代码，通常会有几个疑问：

  1. 为啥不直接调用 `for (const v of something) {…}` 而是 `const v of something()` 呢？

  2. 调用 `something()` 生成的是个迭代器，但为之前说过，可用 `for…of` 遍历的是一个可迭代的对象是吧？

针对第一个问题，`something` 仅仅是一个生成器，`for…of` 循环可不能遍历它；而第二个问题，generator 返回的迭代器同样定义了 `Symbol.iterator` 的函数，并且它的返回值依旧是 `this`。

#### 停止生成器(Stopping the Generator)
在之前的代码中，当 `*something()` 遇到 `break` 语句后，会一直保留其调用栈么？答案当然是否定的：当`for…of` 循环发生了 "非正常完结(Abnormal completion)" 后 —— 通常是由 `break`、`return` 或 未捕获的异常(uncaught exception) 导致 —— 此时 generator 就会接受到终止的信号。

**Note**：从技术角度讲，当正常的遍历结束是，`for…of` 循环会给迭代器发送一个终止信号。对于 generator 生成的迭代器，这样的结束信号并没有什么用，因为 generator 的迭代器会早一步 `for…of` 自动结束迭代。但是，对于自定义的迭代器，这个正常结束遍历信号，有时候就显得很有价值了。

`for…of` 会自动触发结束信号并发送给迭代器，若是想要手动触发的话，可以调用迭代器上定义的 `return(…)` 方法。

在 generator 中定义的 `try…finally` 在有一个妙用 —— 当触发结束信号的时候，你可以在 `finally` 语句中做一些收尾的工作，比如清除缓存、断开数据库链接啥的：

```js
function *something() {
  try {
    let nextVal;

    while (true) {
      if (nextVal === undefined) {
        nextVal = 1;
      } else {
        nextVal = (3 * nextVal) + 1;
      }

      yield nextVal;
    }
  } finally {
    console.info('cleaning up!');
  }
}
```

在外部手动的调用 `return()` 来结束迭代：

```js
const it = something();
for (const v of it) {
	console.info(v);

	if (v > 500) {
		console.info(it.return('Hello World').value);
	}
}
// 1 9 33 105 321 969
// cleaning up!
// Hello World
```

`it.return(…)` 会立即结束 generator，而后执行 `finally` 里面的代码。传入的值 `'Hello World'` 会直接返回到属性 `value` 中，此时返回的 `done` 为 `true`，因此 `for…of` 会结束遍历。

以上种种，仅仅是 generator 使用的冰山一角。接下去我们要探讨的才是 generator 最核心的使用方式 —— 配合异步使用。

## 生成器异步的迭代(Iterating Generators Asynchronously)
那么，generator 到底解决了异步编程什么问题呢？

先快速回顾下之前我们使用的异步编程方式 —— 回调函数：

```js
function foo(x, y, cb) {
  ajax(
    'http://some.url.1/?x=' + x + '&y=' + y,
		cb
  );
}

foo(11, 31, function (err, txt) {
  if (err) {
    console.error(err);
  } else {
    console.log(txt);
  }
});
```

若是将上面👆的代码改写成 generator 来实现👇的话：

```js
function *main() {
  try {
    var txt = yield bar(11, 31);
    console.log(txt);
  } catch (err) {
    console.error(err);
  }
}

var it = main();

function bar(x, y) {
  ajax(
    'http://some.url.1/?x=' + x + '&y=' + y,
		function (err, data) {
      if (err) {
        it.throw(err);
      } else {
        it.next(data);
      }
    }
  );
}

it.next();
```

第一眼看上去，好像 generator 的实现的异步代码又臭又长。但千万别被表像迷惑，generator 的实现实际上要比回调的实现好很多，就比如对比下面这两段这段代码：

一个 `yield` 就能同步获取到内容：
  ```js
    var txt = yield bar(11, 31);
    console.log(txt);
  ```

而若是直接调用 `foo(…)` 的话，`txt` 却根本没办法拿到网络请求返回的内容：
  ```js
    var txt = foo(
      11, 31,
      function (err, data) {
        //……
      }
    );
    console.log(txt);
  ```

仅仅通过 `yield` 关键字，👆我们就实现了一段看上去像同步代码的异步编程，并且在 `bar(…)` 中屏蔽了异步的细节 —— 这解决了之前我们提及的 *回调函数的执行和我们大脑的工作方式不匹配* 问题。

本质上来看，这一切的实现是基于将异步代码的细节抽象出来，而后我们只用按照 *有序的流* 一样的同步的代码来实现具体细节即可。

### 同步代码的错误处理
既然是以同步的方式来书写的异步代码，那么对于错误的处理，`try…catch` 依然是不二的选择：

```js
try {
  var txt = yield bar(11, 31);
  console.log(txt);
} catch (err) {
  console.error(err);
}
```

这一切能够实现，都源于 `yield` 的暂停机制 —— 不仅能够暂停后续代码的运行，等待异步的回调函数 `return` 的值，而后恢复代码的执行；同样的，对于异步函数里所产生的错误，同样可以以同步的方式获取到；并且，无论这个错误是产生在 generator 函数内，还是函数外：

比如在 generator 函数外，通过迭代对象调用 `throw(…)` 方法，手动触发的错误：

  ```js
    function *main() {
      var x = yield 'Hello Generator';
      console.log(x);
    }

    var it = main();
    it.next();
    try {
      it.throw(42);
    } catch (err) {
      console.error(err); // 42
    }
  ```

又比如，直接在 generator 函数内产生的错误：

  ```js
    function *main() {
      var x = yield 'Hello Generator';
      yield x.toLowerCase(); // x 非字符串，调用 toLowerCase 方法会导致错误
    }

    var it = main();

    it.next();

    try {
      it.next(42); // x 为数字 42
    } catch (err) {
      console.error(err); // TypeError
    }
  ```

  或者在 generator 函数内部就搞定：

  ```js
    function *main() {
      try {
        var x = yield 'Hello Generator';
        var y = yield x.toLowerCase();
      } catch (err) {
        console.error(err); // TypeError
      }
    }

    var it = main();
    it.next();
    it.next(42);
  ```

无论是手动触发的错误，还差 generator 函数自身的错误，通过 `try…catch` 的捕获，在 generator 内部，就有处理错误的机会；但是当其内部没有错误处理的时候，在 generator 函数的外部一定要搞定这些。

在 generator 中，能够通过 `try…catch` 来实现同步的错误处理，在可读性和可维护性上都是一个巨大的胜利。

## Generators + Promises