# Promises
通过前面一节的深入探讨，我们已经很清楚 回调函数 所面临的问题了 —— 书写的反人性和信任问题。显然最为头疼的是由于 *控制反转(inversion of control)* 所带来的信任问题。幸好在 ES6 中已经提供了内置的 API 来解决这个问题 —— Promise。

## Promise是什么？(What Is a Promise?)
许多开发者学习一个新技术的第一步通常是直接撸代码，搞出一个 “Hello World” 什么的。当然这没什么不好，不过想要深入理解 Promise，以及它背后所隐含的理论，不妨先从两个类比开始：

### 未来的值(Future Value)
想象一个在现实世界中常见的场景：当你走进一家快餐店(KFC啥的)的前台，准备买一个汉堡套餐来解决午饭时，你通常是先付钱，而后拿到一张小票，上面有一个数字，代表的就是“我欠你一份汉堡套餐”的 *承诺(promise)* —— 通常情况下，你最终会凭此拿到一份汉堡套餐。

在等待汉堡套餐的时间里，你会干什么呢？打开手机翻翻朋友圈？给某人发个消息说待会吃完中饭去找他聊聊天？或者浏览下最新的热点新闻？……反正不会瞎等在那里就对了 —— 而在这自信满满的等着的汉堡套餐，就是所谓的 **future value**。

时间溜走了，当 “113号，请取餐” 的声音响起，你充满期待的去前台准备接下去的大快朵颐时，此时会有两个情况等着你，第一种会按照你的期待进行，自不必说；另外一种小概率但未必不会发生的事件是：“很抱歉，我们的汉堡卖完了，您看要不来份三明治怎么样？” —— 每次当你想买汉堡套餐时，你都得做好失败的准备。

**Note**：这里的类比是个简化的场景，显然在写代码的时候，要考虑更多的事情，比如你的号码一直都没被叫(很有可能是被忘记了)，这与之对应的就是一直 “悬而未决” 的状态。

#### 现在和将来的值(Values Now and Later)
直接来段代码：

```js
var x, y = 2;

console.log(x + y); // NaN
```

👆 `x + y` 最终产生的是一个与预期结果不符的 `NaN` —— 难道你还指望 `+` 帮你监视着 `x` 和 `y` 都已经是 `number` 类型的值了，而后再帮你完成加法运算？

为了解决这种问题，你得靠自己，比如用回调函数：

```js
function add (getX, getY, cb) {
  var x, y;
  getX(function (xVal) {
     x = xVal;
     if (y != undefined) {
       cb(x + y);
     }
  }),
  getY(function (yVal) {
     y = yVal;
     if (x != undefined) {
       cb(x + y);
     }
  })
}

// `fetchX()` 和 `fetchY()` 是同步或异步的获取对应 x 和 y 值的函数
add(fetchX, fetchY, function (sum) {
  console.log(sum);
})
```

👆先别管它是否优雅，至少它能让行为可控，即在`fetchX()` 和 `fetchY()` 工作正常的情况下，无论它们是同步还是异步，`x + y` 都能正确的返回值期待的值，而非 `NaN`。

#### 承诺的值(Promise Value)
若是把上面的 `add` 用 Promise 来改写的话：

```js
function add (xPromise, yPromise) {
  return Promise.all([xPromise, yPromise])
    .then(function (values) {
      return values[0] + values[1];
    });
}

add(fetchX(), fetchY())
  .then(function (sum) {
    console.log(sum);
  });
```

👆`fetchX()` 和 `fetchY()` 都返回了 Promise 对象，而后被作为参数传入了 `add(…)` 中。

在 `add(…)` 内部，`Promise.all([…])` 接收了它们，随后它也返回一个 Promise 对象，而后在其链接的 `then(…)` 方法中，完成了 `x + y`(`values[0] + values[1]`) 的动作，并作为立即 resolve 的值返回。

最终，在调用 `add(…)` 返回的 Promise 对象后面，链接上 `then(…)` 方法，并将 `sum` 输出到了控制台上。

关键的地方是 Promise 对象本身包含了 `then` 方法，而调用 `then(…)` 方法本身又会返回一个 Promise 对象，因此这个链可以无限进行下去。

和买汉堡套餐的例子一样，我们得对意外的情况做好准备，在 Promise 中，对应的就是 `then(…)` 方法的第二个参数：

```js
add(fetchX(), fetchY())
  .then(
    function (sum) {
      console.log(sum);
    },
    function (err) {
      console.error(err);
    }
  );
```

因为 Promise 是独立于时间的，因此它可以很好的预测和组合各种情况和结果。并且更重要的是，一旦 Promise 的状态 resolve 掉了，就不能更改了 —— 尽管这个状态的结果能够被无限次的访问。

**Note**：这种 *不可更改(immutable)* 的特性，使其能够被任意的传入到各种第三方的代码中，而不必担心被无意或是恶意的篡改。而这是 Promise 设计之初，殚精竭虑完成的最基础、最重要的功能。

Promise 是一种能够很容易的封装、组合 *未来的值(fature value)* 的机制。

### 完成事件(Completion Event)
除了代表 “未来的值” 之外，对 Promise 的另一个洞见是把它视为是一个 “流程控制” 的机制 —— 将异步的任务拆解为多个步骤用来执行。

假设有一个函数 `foo` 会执行一些任务，但我们并不知道它执行的细节，以及它到底是立即执行还是延迟一会后再执行，我们只知道它会通知我们它已经完成，而后我们可以接下去做后续的任务 —— 在 JS 的编程范式中，你通常需要监听这些 “通知”，就好像事件触发的机制一样，我们能够自定义一些需求，打包成一个回调函数，并将回调函数传递给 `foo`；最终 `foo` 会根据情况触发这个回调函数，以此来完成我们的需求。

不过在 Promise 的机制下，这样的关系被反转了 —— 我们的回调函数不再是被 `foo` 触发的，而是通过 Promise 这个中间层来触发，比如下面的一段伪代码：

```js
foo(x) {
	// 异步操作……
}

foo(42)

on (foo, 'success') {
	// foo 完成了，接着做下一步事情
}

on (foo, 'error') {
	// foo 发生了点问题，处理一下
}
```

`foo(…)` 不必关心，也不用要处理一些订阅它的事件，这些都交给了 `on(…)`，这是一种很好的 "责任分离(separation of concerns)" 的实现。

但在 Promise 出来之前，在 JS 中想要实现这样的效果，只能 “魔改” 一把：

```js
function foo(x) {
	// 异步操作……

	// 返回一个 listener 对象，它有一个 `on` 方法，能够实现事件的绑定
	return listener;
}

const evt = foo(42);

evt.on('success', function(){
	// foo 完成了，接着做下一步事情
});

evt.on('error', function(err){
  // foo 发生了点问题，处理一下
});
```

👆 `foo(…)` 返回了一个对象能够处理事件的订阅，如此一来我们就能像事件机制一样来完成对 `foo` 后续任务的处理。但说到底，`listener` 依然是由 `foo` 内部实现，并没有颠覆 “控制反转” 的底层逻辑。

当然，你还能将监听 `foo` 的 `success` 和 `error` 事件分开，单独封装成 `bar` 和 `baz`：

```js
// `bar` 监听 `foo` 的 success
bar(evt);

// `baz` 监听 `foo` 的 error
baz(evt);
```

若是 `evt` 或者说 `listener` 是由一个可靠的第三方机制来生成的话，那的的确确实现了 “反控制反转(Uninversion of control)”，并且加上 “责任分离” 的效果，看上去真的很 nice。

#### Promise“事件”(Promise "Events")
其实 `evt` 就是对 Promise 的一个类比，在 Promise 的机制下，`foo(…)` 会返回一个 Promise 的实例，而后这个实例会作为参数传递给 `bar(…)` 和 `baz(…)`。

**Note**：需要注意的是之前对于事件的措辞，严格来讲，Promise 的实现并不是通过事件机制，更没有 `success` 和 `error` 之类的事件监听；不过你也能将调用其实例上的 `then(…)` 和 `catch(…)` 方法，理解成注册了 `fulfillment` 或 `rejection` 事件 —— 虽然我们不会在真实的代码中展示出来：

```js
function bar (fooPromise) {
	fooPromise.then(
		function(){
			// resolve 监听
		},
		function(){
			// reject 监听
		}
	);
}

// baz 同上

function foo(x) {
	// 异步操作……

	// 返回一个 Promise 的实例
	return new Promise(function(resolve, reject){
		// 内部调用 `resolve` 或则 `reject`，最终会触发绑定的回调
	});
}

var p = foo(42);

bar(p);

baz(p);
```

👆 `new Promise(function (resolve, reject) {})` 这种模式被称为 [revealing constructor](https://blog.domenic.me/the-revealing-constructor-pattern/) —— 被传入 Promise 构造函数的函数会被立即执行，并接受两个参数，一个是 `resolve`，调用它会产生 fulfillment 的状态；另一个是 `reject`，调用它会产生 rejection 的状态。

另一种更优雅实现的方式，将操作 `resolve` 和 `reject` 的代码分离，而不是耦合在一起：

```js
function bar() {
	// resolve 监听
}

function oopsBar() {
	// reject 监听
}

// `baz()` 和 `oopsBaz()` 同上

var p = foo(42);

p.then(bar, oopsBar);

p.then(baz, oopsBaz);
```

**Note**：`p.then(…).then(…)` 和 `p.then(…);p.then(…)` 是两个完全不一样的代码，前者的链式调用中，前后两个 `then(…)` 是不同的 Promise 实例上的方法；后者则都是出自同一个 Promise 实例的 `then(…)` 方法，并且它的状态一旦确定，就无法被改变了。


## Then的鸭式类型(Thenable Duck Typing)
使用 Promise 面临的最头疼的问题是如何确定某个对象是不是真正的 Promise。

`p instanceof Promise` 这样的检测方式没办法完全满足需求的主要原因是，浏览器窗口之间(iframe)的交互，若是它们使用的 Promise实例 是不同的构造函数创建的，那这样的检测就会失败。

再者，若是你使用了某个三方的库，它的 Promise 并非是基于原生的 Promise 来实现，那么也无法满足需求。

因此，社区内广泛流行的一种 类型检测 机制是基于 “鸭式类型(duck typing)” —— 如果它长得像鸭子，并且能像鸭子一样呱呱的叫，那么它就是鸭子。这种基于 “值的形状” 的检查，被用于 “Promise的真假美猴王” 之中，即所谓的 thenable：

```js
function isPromise (p) {
  if (p !== null && (typeof p === 'object' || typeof p === 'function') && typeof p.then === 'function') {
    // thenable!
    return true;
  }
  return false;
}
```

好像还行？立马露馅：

```js
var o = { then: function() {} };

var v = Object.create(o);

v.hasOwnProperty('then'); // false

isPromise(v); // true
```

而若是有人 无意/故意/恶意 在一些内置的类的原型对象上面添加了 `then` 方法的话：

```js
Object.prototype.then = function(){};
Array.prototype.then = function(){};

var v1 = { hello: 'world' };
var v2 = [ 'Hello', 'World' ];

isPromise(v1); // true
isPromise(v2); // true
```

👆要是你把这些对象作为参数传递给之前的 `baz`、`bar` 的话，那它们会被永远的挂起……

你可能会觉得，这是危言耸听吧？！遇到这种情况的概率实在是太小了。但你不得不防的是有一些你依赖的第三方的库，它们很可能是在 ES6 之前的产物，并且有 `then` 方法挂载在它的某个对象上。遇到这种情况，`isPromise` 就不好使了。

## Promise 和 信任问题(Promise Trust)
虽然前面我们已经通过两个很形象的类比(“未来的值”和“完成的事件”)，了解到 Promise 如何在异步领域大放光彩的，但如果仅仅到这里，不继续深入的话，那就失去了让我们了解其最重要的核心特征(也是Promise 之所以存在的根本原因)的机会 —— 信任(Trust)。

回顾下基于 回调函数 的异步代码由于其 控制反转 而带来的各种 “信任” 问题：

  - 过早的调用回调函数

  - 过迟的调用回调函数

  - 太少/太多 地调用了回调函数

  - 调用回调函数时，缺少必须的参数

  - 把各种异常和错误都 “吞掉了” 

下面，让我们一个个的来看 Promise 到底是如何解决这些问题的：

### 过早的调用(Calling Too Early)
回调函数可能是同步，也可能是异步被调用，而没有一个内置的机制确保其一定是异步的，除非手动的使用 `setTimeout(…)`。

而在 Promise 的机制中，任何放到 `then(…)` 中的回调函数，能够被确保总会是异步被调用的(准确来讲是通过 *任务队列* 的机制来实现的)。

### 过迟的调用(Calling Too Late)
👆 和上面的一样，任何放到 `then(…)` 中的回调函数，在 调用了由 Promise 提供的 `resolve(…)` 或 `reject(…)` 后，能够确保总会在接下去的任务队列中被调用 —— 你不可能在同步的任务中观察到它被执行。因此，当一个 Promise 实例的状态发生改变，那么所有注册在它的 `then(…)` 方法上的回调函数都会按照注册的顺序被调用，即便是在这些回调函数的内部，也没办法影响(延迟)其他的回调函数：

```js
const p = new Promise((resolve, reject) => {
  resolve('some data');
});

p.then(function () {
  p.then(function () {
    console.info('C');
  });
  console.info('A');
});

p.then(function () {
  console.info('B');
});

// A B C
```

👆 `'C'` 无法阻止或改变 `'B'` 的执行(顺序)，而这也是 Promise 的优势之一。

#### Promise 顺序的怪癖(Promise Scheduling Quirks)
需要格外注意的是，在两个分开的 Promise 链式代码里的回调函数，你是没有办法确保它们的执行顺序的。比如，两个 Promise 的实例 `p1` 和 `p2` 同时变到了 resolved 的状态，`p1.then(…)` 和 `p2.then(…)` 书写的先后顺序通常是其内部回调函数的执行顺序，但若是改变一些细节：

```js
const p3 = new Promise((resolve, reject) => {
  resolve('B');
});

const p1 = new Promise((resolve, reject) => {
  resolve(p3);
});

const p2 = new Promise((resolve, reject) => {
  resolve('A');
});

p1.then(function (v) {
  console.info(v);
});

p2.then(function (v) {
  console.info(v);
});

// A B
```


![Promise Scheduling Quirks](./assets/promise_schedule_quirks.jpg)


👆并非按照设想的打印出了 `B A`，这是因为 `p1` 中的 `resolve(p3);` 其实是 resolve 了另一个异步任务，而 `p2` 则 resolve 的一个同步任务，因此在这个维度看，`p2` resolve 的结果要早于 `p1` 的结果。不过这一切都发生在同一个任务队列中，即 **它们resolve的结果** 并没有同步和异步的区别。

### 没有调用回调函数(Never Calling the Callback)
这个问题在 Promise 中很好搞定：只要你注册了 fulfillment 和 rejection 的回调函数，一旦 Promise 的状态改变，就必然会调用其中的某一个回调。至于所谓的会被 “永远地挂起”，即状态永远没有任何改变？其实很好解决：

```js
function timeoutPromise (delay) {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      reject('timeout');
    }, delay);
  })
}

Promise.race([
  foo(),
  timeoutPromise(3000)
])
.then(
  function () {},
  function (err) {}
);
```

一旦 `3000` 毫秒过了，`foo()` 返回的 Promise 实例还没确定状态的话，就会走到超时的逻辑中，因此你也不用担心你的程序会被 “永远地挂起”。

### 太少/太多 的调用(Calling Too Few or Too Many Times)
理想的回调函数被调用的次数肯定是 one —— 这没有疑议吧？！那么 “太少” 就是没有被调用，那就回到👆上面提到的部分。

“太多” 的疑虑也很容易被解决 —— 一旦 Promise 状态确定了，那之后就无法再次改变，即便你反复的调用实例化时，提供给你的 `resolve(…)` 和 `reject(…)`，Promise 实例只会接受第一个被调用的，其他的都会被忽视。

因此，任何注册在 `then(…)` 中的回调，只会被调用一次。当然，你能够无限制的在相同的 Promise 实例中注册相同的回调函数，比如 `p.then(f); p.then(f);`，那它们会被与之对应的次数… —— 不过那是你自己的非得要搬起石头砸自己的脚，你怪谁？！

### 缺少所需的参数(Failing to Pass Along Any Parameters/Environment)

`resolve(…)` 和 `reject(…)` 都能接收 **一个** 任何类型的参数，它们会被传递给所有注册的回调函数。

若是超过了一个参数？除了第一个，其他都会被 “无情的抛弃”。千万别幻想用 `resolve(…);resolve(…);resolve(…);……` 来实现多个参数的传递 —— 还是老老实实的用 `object` 或者 `array` 来包裹多个参数比较现实。

另外，任何注册到 `then(…)` 中的回调函数，都保存了定义它时的上下文，即 “闭包” 这个能力依然有效。

### “吞掉了”各种异常和错误(Swallowing Any Errors/Exceptions)

```js
const p = new Promise(function (resolve, reject) {
	foo.bar();
	resolve(42);
});

p.then(
	function fulfilled () {
    // 永远到不了这儿
  },
	function rejected (err) {
    // TypeError
  }
);
```

👆 `foo.bar();` 是产生错误的源头，而且即便是这个异常是发生在 Promise 的实例化过程中(同步)，也会让这个异常在异步的行为中被捕获 —— 这一点非常牛逼，能够让你无感的就解决了之前提到的 race condition 的问题，无论这段代码否产生了异常。

那若是异常发生在 `fulfilled` 的回调函数中呢？

```js
const p = new Promise(function (resolve, reject) {
	resolve(42);
});

p.then(
	function fulfilled (val) {
    foo.bar();
    console.info(val);
  },
	function rejected (err) {
    // 永远到不了这儿
  }
);
```

好像把异常“吞掉了”？那只是幻觉 —— `p.then(…)` 其实会返回另一个 Promise 的实例，而这个实例则会进入到异常捕获的状态。

你可能会想，为啥不直接进入到当前这个 Promise 实例的异常捕获完事？当然不能，因为这违背了 Promise 最重要的原则 —— 一旦状态确定就不能改变。即 `p` 总是 值为 `42` 的 fulfilled 的状态。而若是违背了这一原则，试想下，当你在同一个 Promise 的实例上，注册了多个回调的时候，就会发现有的结果是这样，有的结果是那样……

### 能被信任的Promise?(Trustable promise?)
在我们完全信任 Promise 之前，还有最后一个问题需要仔细的探讨下 —— Promise 并没有摆脱掉回调函数，而仅仅是改变了回调函数传递的位置 —— 不再是把回调函数传到 `foo` 里面去，而是将回调函数传入 `foo(…)` 的返回值中。但为什么这种模式就值得信任呢？难道仅仅是因为它已被信任了，所以就信任了么？！

对于内置的静态方法 `Promise.resolve(…)`，若是你传入的是一个非异步、非Promise、非thenable的值，那你会得到一个 `fulfilled` 的值，比如👇下面两个 Promise 其实是没啥区别的：

```js
var p1 = new Promise((resolve, reject) => {
  resolve(42);
});

var p2 = Promise.resolve(42);
```

但若是你传入的是一个 Promise 的值，那则会原封不动的将其返回：

```js
var p1 = Promise.resolve(42);

var p2 = Promise.resolve(p1);

p1 === p2; // true
```

![promise_resolve](./assets/promise_trusable_promise.png)

更为重要的是，对于随意使用 非Promise的thenable 导致的意外的后果：

```js
var p = {
	then: function(cb, errcb) {
		cb(42);
		errcb('evil laugh!');
	}
};

p
.then(
  function fulfilled (val) { console.log(val); },
  function rejected (err) { console.error(err); }
);
```

![thenable](./assets/promise_trusable_promise_thenable.png)

👆 `p` 是一个 `thenable` 但并非是严格按照 promise 的规范来设计的，它不能被信任。不过呢，我们可以把它作为值，传入到 `promise.resolve(…)` 中，来解决信任问题：

```js
Promise.resolve(p)
.then(
  function fulfilled (val) { console.log(val); },
  function rejected (err) { console.error(err); }
);
```

![thenable_resolved](./assets/promise_trusable_promise_thenable_resolve.png)

因此，若是当我们要使用某个第三方的 thenable 的工具函数 `foo(…)` 时，`Promise.resolve(…)` 能确保其返回的值一定是可信任的 Promise：

```js
Promise.resolve(foo(42))
.then(function (v) {
	console.log(v);
});
```

除此之外，`Promise.resolve(…)` 还有个好处它能 “格式化” 传入的值 —— 无论传入的值是立即执行的同步代码，还是延迟执行的异步代码，最终它返回的值都是异步的。

### 信任的建立(Trust Built)
回到多年前，当 JS 的所有异步任务只能用回调函数解决的时候，信任问题有显得那么重要么？不一定吧！？但是时至今日，当你一旦接触而后使用过了 Promise，再回头看看以前的，仅仅由回调函数构成的摇摇欲坠的异步代码时，你发现你再也回不去了……

Promise 通过颠覆控制反转，带来的不仅仅是可信任的代码和执行机制，更为重要的是它解放了我们的心智，让我们能够更专注于其他任务。

## 链式流(Chain Flow)
前面我们已经看过不止一遍了，我们可以用一连串 *this-then-that* 的操作，将许多个 Promise 串联起来，组成一个有序的异步链式代码块。

而让这一切起作用的核心，源自于 Promise 的两个内置的行为模式：

- 每次调用 Promise 的 `then(…)` 方法后，都会返回一个新的 Promise，因此我们可以一直链接下去；

- 无论 `then(…)` 的 fulfillment 回调函数(即 `then` 的第一个参数) return 任何值，它都会被自动的设置为 Promise 链中的 fulfillment 状态。

```js
var p = Promise.resolve(22);

var p2 = p.then(res => {
  console.log(res); // 22
  return res * 2;
});

p2.then(res => console.log(res)); // 44
```

return 的 `res * 2` 就是第一个 `then(…)` 方法的 fulfillment 回调函数返回的值，它会被自动设置为 `p2` 的 `then(…)` 方法的 fulfillment 回调函数接收到的值。而后你可以将 `p2.then(…)` 返回的值储存到另一个变量 `p3` 中，并操作它的 `then(…)` 方法……

可即便是将 Promise 储存于 `p2` 也显得有些多余，直接 “串” 起来其实就可以了：

```js
var p = Promise.resolve(22);

p
.then(res => {
  console.log(res); // 22
  return res * 2;
})
.then(res => console.log(res));
```

👆无论你想 “串” 起多少个 `then(…)` 都是可以的，这源于每个 `then(…)` 都会自动的返回一个新的 Promise 的本质所在。

到这里，“这道菜” 似乎一切都很完美，但还是感觉缺点调料 —— 若是想在 `then(…)` 中处理一个异步事件，发起一个网络请求的话，要怎么做呢？关键依然在 `then(…)` 的返回值上：

```js
var p = Promise.resolve(22);

p
.then(res => {
  console.log(res); // 22
  return new Promise((resolve, reject) => {
    resolve(res * 2); // 将 Promise 的状态设置为 fulfill，以便能在下一个 then(…) 中的第一个回调函数中拿到相应的值
  });
})
.then(res => console.log(res));
```

👆看见了么？关键是 return 一个 thenable 或 Promise，而后在下一个 `then(…)` 方法里的回调函数被调用前，会一层层地递归拆解(unwrapping)这个返回的 thenable 或 Promise，直到它的最后状态被确定下来(resolution)为止。

因此，想要在这一串链式中实现异步操作：

```js
var p = Promise.resolve(22);

p
.then(res => {
  console.log(res); // 22
  return new Promise((resolve, reject) => {
    // 1000 毫秒后的异步操作
    console.time();
    setTimeout(() => resolve(res * 2), 1000);
  });
})
.then(res => {
  console.timeEnd();
  console.log(res);
}); // log会发生在 1000 毫秒后
```

![promise_async](./assets/promise_chain_flow_async.png)

👆这也意味着，每一步都能使用异步，精准地控制何时触发下一步的回调函数，形成一条完整的异步流：

```js
function delay (time, data) {
  return new Promise(function (resolve, reject) {
    setTimeout(() => resolve(data), time)
  });
};

delay(100)
.then(function step2() {
  console.log('step 2 after 100ms');
  return delay(200);
})
.then(function step3() {
  console.log('step 3 after 200ms');
})
.then(function step4() {
  console.log('step 4 next job');
  return delay(50);
})
.then(function step5() {
  console.log('step 5 after 50ms');
})
```

若是没有指定 return 值，那么就会返回 `undefined`，但这并不会影响到 Promise 的链式流。但说句实话，简单的使用 `setTimeout` 来延迟代码执行的时间，在实际的开发中并没有什么用，👇以下是更贴近实际开发的例子：

```js
function request (url) {
  return new Promise((resolve, reject) => {
    ajax(url, resolve);
  });
}

request('http://some.url.1/')
.then(response1 => request(`http://some.url.2/?v=${response1}`))
.then(response2 => console.log(response2));
```

两个串行的网络请求，第二个请求依赖于第一个请求返回的值，这样的场景在实际的开发过程中很常见。此时 Promise 的链式结构不仅实现了多个异步任务按照顺序串行进行，而且还能够在每个步骤间实现消息的传递。

不过，任何程序都可能会发生异常情况，在 Promise 中当然也内置了对异常的处理逻辑：

```js
request('http://some.url.1/')
.then(response1 => {
  foo.bar(); // error!
  request(`http://some.url.2/?v=${response1}`);
})
.then(
  response2 => console.log(response2),
  err => {
    console.log(err);
    return 42;
  }
)
.then(msg => console.log(msg)); // 42
```

`then(…)` 的第二个参数就是当异常发生时，被调用的 rejection 回调函数。无论是 `reject`、`throw new Error(…)`、或者是上面看到错误代码 `foo.bar();`，都会激活 rejection 回调函数。

无论是忽略 fulfillment 还是 rejection 的回调函数，一旦 Promise 的状态被确定后，都会继续沿着 Promise 链冒泡，直到遇见处理它的回调函数为止：

```js
var p = Promise.reject('reject msg!');

p
.then(
  res => console.log('fulfillment handler: ', res)
)
.then(
  null,
  err => console.log('rejection handler: ', err)
);
```

![rejection_handler](./assets/promise_chain_flow_rejection.png)

**Note**：本质上来看，`then(null, function (err) { //... })` 的模式和 `catch(function (err) { //... })` 行为一致。

无论链式流多么有用，都不该把它视为是 Promise 的最核心的意义所在，顶多算作是一个副产品。而 *标准化异步操作流程* 和 *将时间和状态都封装到值的依赖中*，是让我们能够将其链接起来的关键所在。

不过，链式调用的 *this-then-this-then-this...* 的模式依然有太多的 “八股(boilerplate)” 需要一次又一次的被实现出来，而解决这个问题是需要一个更为优雅的异步流程控制模式 —— generator。

### 术语：Resolve, Fulfill 和 Reject(Terminology: Resolve, Fulfill, and Reject)
对于 "resolve"、"fulfill"、"reject" 这些术语，好像大致能区别它们，但始终有一些困惑，不明白它们之间到底有啥不一样。先别急，来看一段代码：

```js
var p = new Promise(function (x, y) {
  // x() -> fulfillment
  // y() -> rejection
});
```

通常情况下，实例化 Promise 的回调函数中，第一个参数的调用 `x()` 标志着 Promise 被定为 fulfill 的状态，第二个参数则代表 Promise 被定为 reject 的状态。但请注意这个细节 —— “通常”。

本质上来看，你写的代码 `function (x, y) { //… }` 对于 JS引擎 来说都是一样的 —— 无论是 `x` 还是 `y` 对它来讲都只不过是相同的函数体，但是函数的名字，不仅仅关乎你自己对于代码的理解，更为关键的是你团队的其他开发者如何理解你的代码。一段即便是精心编排设计，但思考错误的异步代码带来的危害，远比意大利面条般的回调地狱(spaghetti-callback)异步代码危害更大，因此如何给 `x` 和 `y` 一个准确的命名显得尤为重要。

👆使用 `reject(…)` 来命名实例化 Promise 回调的第二个参数是一个正确的选择。而对于第一个参数的称谓，即便是官方推荐使用的 `resolve(…)`，却也总是会让人联想到 "resolution(决定)" 的意思 —— 这个含义通常会被理解为 “确定 Promise 最终的值/状态”。

若是第一个参数被指定为 Promise 的 fulfill 状态的话，那为何我们不用 `fulfill(…)` 取代 `resolve(…)` 呢？要回答这个问题，还得仔细看看两个 Promise 的 API：

```js
var fulfilledPr = Promise.resolve(42);
var rejectedPr = Promise.reject('err');
```

`Promise.resolve(…);` 这个 API 将数字 `42` 定为 Promise 的最终值，状态为 fulfilled；而 `Promise.reject('err');` 显然是创建一个状态为 rejected 的 Promise，理由(值)是 `err`。

👆这好像并没有说明为啥 `resolve(…)` 不用 `fulfill(…)` 的问题？先别急，看下面一段代码：

```js
var rejectedTh = {
  then: function (resolved, rejected) {
    rejected('err msg');
  }
};

var rejectedPr = Promise.resolve(rejectedTh);
```

还记得吗？`Promise.resolve(…)` 会将传入其中的 thenable 或者 Promise 解析并展开，就上面的例子而言，`Promise.resolve(…)` 会返回一个状态是 rejected 的 Promise。因此，这个时候若是用 `fulfill(…)` 作为方法名就显得很不合时宜了，相比较而言，`resolve(…)` 显然是更好的选择。

而即便是在实例化 Promise 的回调函数中，第一个参数同样和 `Promise.resolve(…)` 一样，会解析传入其中的 thenable 或者 Promise：

```js
var rejectedPr = new Promise(function (resolve, reject) {
  resolve(Promise.reject('err msg'));
});

rejectedPr.then(
  function fulfilled () {
    // never gets here
  },
  function rejected (err) {
    console.log(err); // "err msg"
  }
);
```

所以，`resolve(…)` 显然也是实例化 Promise 的回调函数中的第一个参数更为准确的名称。

**Note**：👆 `resolve(Promise.reject('err msg'));` 并不会把 `Promise.reject('err msg')` 的 `'err msg'` 解析出来，做这件事情的是 `then(…)` 中的 `rejected` 回调函数。

而对于在 `then(…)` 方法中定义的两个回调函数呢？`fulfilled(…)` 和 `rejected(…)` 显然更为准确：

```js
function fulfilled (res) {
  console.log(res);
}

function rejected (err) {
  console.error(err);
}

p.then(
  fulfilled,
  rejected
);
```

在 ES6 的说明文档里面用的是 `onFulfilled(…)` 和 `onRejected(…)`，而实际上也的确如此，第一个回调的触发是 Promise 被确定为 fulfill 状态；而第二个回调的触发是 Promise 被确定为 reject 状态。


## 错误处理(Error Handling)
同步的 `try…catch` 是大多数JS开发者选择的异常处理方式。但是在异步代码中发生的异常，这个方案显得力不从心：

```js
function foo () {
  setTimeout(function () {
    baz.bar();
  }, 1000);
}

try {
  foo();
}
catch (err) {
  console.error('try catch 捕获的异常：', err);
}
```

除非通过一些特殊的手段，否则 `try…catch` 没办法搞定异步代码的异常处理。

若是传统的回调异步，异常的处理一般是通过所谓的 “错误优先(error-first callback style)” 模式：

```js
function foo (cb) {
  setTimeout(function () {
    try {
      var x = baz.bar();
      cb(null, x);
    }
    catch (err) {
      cb(err);
    }
  }, 1000);
}

foo(function (err, val) {
  if (err) {
    console.error('error-first callback: ', err);
  } else {
    console.log(val);
  }
});
```

传入 `foo(…)` 的回调函数的第一个参数，若是非 falsy 的值，就会被认为异常发生了；否则的话，就是成功了。虽然这种异步的错误处理在技术上是可行的，但是和回调地狱的问题一样，若是有多个 `if…else…` 的面条代码，绕也能把你绕晕。

Promise 采用的异常处理策略是 “分而治之”，即将异常处理的回调函数拆分开：

```js
var p = Promise.reject('err msg');

p.then(
  function fulfilled () {
    // …
  },
  function rejected (err) {
    console.error(err);
  }
);
```

这种模式看上去好像挺好的，但其实呢，也是有隐含的问题存在的：

```js
var p = Promise.resolve(42);

p.then(
  function fulfilled (res) {
    // number 没有 toLowerCase 方法
    // 因此会抛出异常
    console.log(res.toLowerCase());
  },
  function rejected (err) {
    // …
  }
);
```

👆之前我们[讨论过](https://github.com/BobbyLH/ReadingNotes---You-Dont-Know-JS/blob/master/async%20%26%20performance/Promises.md#%E5%90%9E%E6%8E%89%E4%BA%86%E5%90%84%E7%A7%8D%E5%BC%82%E5%B8%B8%E5%92%8C%E9%94%99%E8%AF%AFswallowing-any-errorsexceptions)为什么 `function rejected (err) { // … }` 没办法捕获到异常，这里就不再赘述。

这很好的解释了为什么在 Promise 中处理异常很容易出问题。

**警告**：若是你在创建 Promise 的过程中，没有按照规范来(比如 `new Promise(null)`、`Promise.all()`、`Promise.race(42)`)，那这个异常会立刻被抛出，而不是被解析为 rejected 状态后，由 reject 的回调函数来处理。

### 绝望之坑(Pit of Despair)
Jeff Atwood 多年前的一篇博客 [绝望之坑](https://blog.codinghorror.com/falling-into-the-pit-of-success/) 是这一节标题的起源。

而 Promise 也有所谓的 “绝望之坑” —— 对错误处理 —— 一旦你忘记去监听它的错误事件，那它默认就会吞噬掉所有的错误信息，这显然难以让人接受。

针对这个问题，社区中提出的避免的办法是：无论如何，一定要在你的 Promise 代码后加上 `catch(…)`，以此来保证错误信息能被捕获：

```js
const p = Promise.resolve(42);

p
  .then(msg => console.log(msg.toLowerCase()))
  .catch(handleErrors);
```

👆在 `then(…)` 方法中的 `msg.toLowerCase()` 显然会导致错误的发生，因为 `p` resolve 的值为数字类型的 `42`，而紧随其后的 `catch(…)` 在这个例子中能够很好的捕获到错误的信息。

所以，搞定了？没那么简单 —— 若是 `handleErrors(…)` 自己本身发生了错误呢？这就好像监管者自己不遵守规则，从而监守自盗一样，而你也没办法通过再加一个 `catch(…)` 来解决问题，因为，这个监控程序依然可能发生错误 —— 好像进入了死胡同？

### 未捕获的异常(Uncaught Handling)
目前为止，还没有办法能够彻底解决上面的问题，但不排除的确有办法能够处理的更优雅一些。

有一些 Promise 的第三方库实现了一个 “全局未捕获异常(global unhandled reject)” 的处理逻辑 —— 比方说当监测到 Promise 的状态变成了 rejected，此时开启一个定时器，在 3 秒后若没有任何的异常处理程序被触发，那么它就会假定说这个 Promise 有一个 `uncaught` 的异常。

但，通过 3 秒来判断是否发生了 “未捕获的异常” 显然太过武断了，即便是这数值(3秒)通过了大量的实验验证，比如依然会有这样的情况存在：在某个场景下发生的错误，就是需要你延时处理，这个时候你希望的是 Promise 能 hold 住这些错误，而不是报一个 uncaught errors。

另外的一种呼声是在 Promise 中添加 `done(…)` 方法：在这个方法执行完后，它不会像其他方法一样返回一个新的 Promise，这就会结束到无休止的链式代码，因此由链式调用可能导致的无穷尽的错误就能够被避免。

而这最终会导致在 `done(…)` 中产生的错误被抛到全局：

```js
const p = Promise.resolve(42);

p
  .then(msg => console.log(msg.toLowerCase()))
  .done(null, handleErrors);
```

👆这比起之前的法子，确实更优雅了。但问题是，这个方案并没有在 ES6 中被标准化，因此最好的出路是找到一个可靠的且通用的解决方案。

还有一种方案，借助浏览器的底层能力：浏览器拥有的底层能力，比起我们能通过JS代码是现实的，要强大很多，就比如对 Promise 调用栈的追踪。当垃圾回收触发时，浏览器当然 Promise 的状态是否是 rejection，而且还能知道是否有 uncaught error —— 把它输出到控制台应该没有任何难度。

但也有问题，比如当 Promise 没有被垃圾回收的时候 —— 这样的情况时长会发生在你的代码越写越多，进而变得杂乱不堪时，鬼知道哪段代码形成了闭包，进而导致内存溢出啥的。那通过浏览器追踪 Promise 的调用栈，从而发现 uncaught error 的办法，就不太靠谱了。

那么，还有别的办法吗？当然！

### 成功之坑(Pit of Success)
虽然下面的部分都是纯理论的，但未来的某天，Promise 说不定就会实现这些功能(即便没有实现，也能通过各种 polyfill 来实现，因为它没有打破任何现有的机制，从而有不错的兼容性)：

- Promise 会在下一轮的 任务(Job) 或 事件循环(event lop) 将任何未捕获的错误报告出来；

- Promise 的实例上会实现一个 `defer()` 方法，它可以无限期的延迟 Promise 未捕获的将要自动上报的错误。

若是 Promise 发生了异常，被 rejected，那它的默认行为是将错误信息打印在控制台而不是将错误 “沉默的吞噬掉”：

```js
const p = Promise.reject('error msg').defer();

foo(42)
  .then(
    function fulfilled() {
      return p;
    },
    function rejected(err) {
      // handle `foo(..)` error
    }
  );
	
```

👆`defer()` 推迟了错误的处理，此时并不会报告任何错误的发生，而且它还会返回一个 Promise 以便于链式调用的进行。

在 `foo()` 的返回的 Promise 的 `then(…)` 方法中，`fulfilled(…)` 返回的 `p` 没有再次 `defer()`，所以会立马将错误信息作为一个 uncaught error 打印到控制台上。

这样的设计是成功的 —— 错误信息要么被捕获处理，要么被打印到控制台出来 —— 这符合大多数开发人员的预期，而且你也能推迟错误的处理，就比如 `Promise.reject('error msg').defer()`。

而唯一的风险来自 `defer()` —— 当你延迟了错误的处理后，你居然最终忘记了处理 —— 你怪谁呢？

## Promise 模式(Promise Patterns)
除了很熟悉的 Promise 链式调用的模式之外，还有另外几种模式能够简化异步编程的流程控制。比如，有两种已经被原生 Promise 支持的模式，常被用来构建异步程序。

### Promise.all([…])
在异步链式流中，一个步骤(step2)任务只能等到之前的步骤(step1)完成后，才能进行；而若是想要同时执行多个步骤中的任务，即并发(平行)呢？

在编程术语中，**gate 机制** 被用来形容某个任务的前置任务是有两个及以上的任务需要执行，至于它们最终完成的顺序不用考虑，只需要在它们全部完成时，执行下一个任务。此时若能同时并发的执行这些前置任务，那就能很好的提高效率。

在 Promise 的API中，这个机制实现就是 `all([…])` 方法。

比如的你业务代码里有三个 ajax 请求，其中的一个请求需要等前面两个请求获取到返回值才能进行，而另外两个请求是独立互不干扰的，这个时候更高效的做法是同时将这两个独立的请求发送出去，而不是一个个的发送：

```js
const p1 = request('https://some.url.1');
const p2 = request('https://some.url.2');

Promise.all([p1, p2])
  .then(function fulfilled (msgs) {
    return request('https://some.url.3/?v=' + msgs.join(','));
  })
  .then(function (msg) {
    console.info(msg);
  });
```

👆 `Promise.all([…])` 接受一个数组参数，在其中的每一项都是由一个 Promise 的实例构成，而它们中的每一个 Promise 返回的值都会按照一开始在数组中的顺序，被纳入到一个新的数组中，最终作为 `fulfilled(msgs)` 的参数。

`Promise.all([…])` 的数组参数可以接受 Promise、thenable、甚至是一个直接值(immediate value)，但它们最终都会被丢进 `Promise.resolve(…)` 包裹一层，被标准化成 Promise 对象。而若是这个数组参数是一个空数组，那么就会立即得到 fulfill 的状态。

只当数组参数中全部的 Promise 都 fulfilled 时，`Promise.all([…])` 才能变成 fulfill 的状态，否则只要有一个 Promise 是 rejected，那 `Promise.all([…])` 就会变成 reject 的状态。

Promise 的异常情况的处理一定别忘记，特别是对于 `Promise.all([…])`。

### Promise.race([…])
有时候，你只想接受第一个状态确定(resolve)的 Promise，而其他的统统抛弃，对于这种情况，编程中的术语叫做 **latch 机制**。对应的，在 Promise 中的实现是 `race([…])` API。

**警告**：千万别把 `Promise.race([…])` 和 race condition 搞混淆了，后者通常是指程序的 Bug。

`Promise.race([…])` 接受的数组参数和 `Promise.all([…])` 相同。但若是某个参数是一个非异步的直接值，那它总会是第一个抵达 latch 的值 —— 这就好像一场跑步比赛中，有一个选手是直接站在终点出发的一样。

在 `Promise.race([…])` 的数组参数中，第一个被确定的 Promise 的状态，就是 `Promise.race([…])` 的状态。而若是参数传入了一个空数组，这会导致 `Promise.race([…])` 永远无法被 resolve —— 一直是 pending 的状态 —— 这个陷阱需要警惕。

将之前的例子中的 `Promise.all([…])` 替换成 `Promise.race([…])`：

```js
const p1 = request('https://some.url.1');
const p2 = request('https://some.url.2');

Promise.race([p1, p2])
  .then(function fulfilled (msg) {
    return request('https://some.url.3/?v=' + msg);
  })
  .then(function (msg) {
    console.info(msg);
  });
```

👆 因为 `Promise.race([…])` 只有一个赢家，因此返回值肯定不是数组，而是单个的值。

#### Timeout Race
`Promise.race([…])` 可以用来实现 "promise timeout" 的编程模式：

```js
function timeoutPromise (timeout = 1000) {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      reject('timeout!');
    }, timeout);
  })
}
Promise.race([
  request('https://some.url.1'),
  timeoutPromise(3000)
])
  .then(
    function () {},
    function (err) {
      console.error(err);
    }
  );
```

👆 实际上，用 `Promise.all([…])` 替换 `Promise.race([…])` 效果是一样的。虽然这个超时的模式能够在大部分情况下都正常工作，但也有一些微妙的细节需要考量。

##### "Finally"
关键的问题是：“在超时后，被抛弃/忽视的那些 Promise 要怎么处理呢？” —— 问这个问题不是为了性能优化，毕竟垃圾回收也不是吃白饭的，问这个问题是考虑到行为的一致性，就比如某些 Promise 有外部的副作用，或者占用了某些资源，而这些 Promise 又不能被取消 —— 这是也为了确保 Promise 外部不可更改的信任机制 —— 因此它们只能安静地被忽视掉。

`finally(…)` API很好的解决了这个问题，它会在 Promise 的状态 resolve 后执行，你可以执行一些收尾的工作：

```js
const p = Promise.reject(42);

p
  .then(
    function fulfilled (data) {
      console.info(data);
    }
  )
  .finally(function () {
    // cleanup
  })
  .then(
    function fulfilled (data) {
      console.info('never gets here!');
    },
    function rejected (err) {
      console.error(err);
      return Promise.reject(err)
    }
  );
```

👆 `finally(…)` 返回一个状态确定的 Promise，因此无论是哪个 `fulfilled` 方法，都不会触发。

同时，我们也可以实现一个静态方法 `Promise.observe(…)`，用来观察 Promise 的状态：

```js
if (!Promise.observe) {
	Promise.observe = function (pr, cb) {
		pr.then(
			function fulfilled (msg) {
				Promise.resolve(msg).then(cb);
			},
			function rejected (err) {
				Promise.resolve(err).then(cb);
			}
		);
		return pr;
	};
}
```

`Promise.observe(…)` + 超时模式：

```js
Promise.race([
  Promise.observe(
    request('https://some.url.1'),
    function cleanup (msg) {
      // cleanup
    }
  ),
  timeoutPromise(3000)
]);
```

因为 `Promise.observe(…)` 可以用来无干涉地观察 Promise 的状态，因此能够监控那些由于超时而被抛弃的 Promise。

### all([…]) 和 race([…]) 的变体(Variations on all([…]) and race([…]))
基于 `Promise.all([…])` 和 `Promise.race([…])` 的思想，出现了几个常见变体：

- `none([…])` 和 `all([…])` 很相似，而区别在于前者将 fulfilled 和 rejected 的状态颠倒了 —— 只有当参数中的全部 Promise 都是 rejected 时，`none([…])` 才能是 fulfill 的状态；

- `any([…])` 和 `all([…])` 也很类似，只不过它忽略所有的 rejected，只要有一个 fulfilled 即可；

- `first([…])` 和 `race([…])` 很相似，它会忽略所有的 rejected，同时只接受第一个 fulfilled；

- `last([…])` 和 `first([…])` 基本一样，区别是只有最后一个 fulfilled 才是赢家；

比如实现 `Promise.first([…])`：

```js
if (!Promise.first) {
	Promise.first = function (prs) {
		return new Promise(function (resolve,reject) {
			// loop through all promises
			prs.forEach(function(pr){
				// normalize the value
				Promise.resolve(pr)
				  .then(resolve);
			});
		});
	};
}
```

👆上面实现的 `Promise.race([…])` 和 `Promise.race([…])` 相比，其不会出现 reject 的状态，而若是你需要处理这样的情况，完全可以自己手动添加相应的处理逻辑。

### 并发迭代(Concurrent Iterations)
通常情况下，迭代一个全部由同步任务构成的 Promise 组成的列表，没啥问题，就好像使用数组中的各种方法(`forEach(…)`、`map(…)`、`every(…)`、`some(…)`、`reduce(…)`……)，但若是要处理一组异步任务，且并发执行的情况，要么你选用第三方的工具，要么自己改造一下 `Promise.all(…)`：

比如作者提供了一个挂载Promise对象下的 `map(…)` 方法，接受两个参数，一个由异步任务组成的数组，另一个是会每次迭代都被执行的回调，最终返回一个状态确定(fullfiled)的数组：

```js
if (!Promise.map) {
  Promise.map = function (vals, cb) {
    return Promise.all(
      vals.map(function (val) {
        return new Promise(function (resolve) {
          cb(val, resolve);
        })
      })
    );
  }
}
```

虽然你无法采用优雅的方式抛出异常(因为只传入了 resolve 作为回调函数的参数)，但若是在回调函数中发生了异常，那么整个 `Promise.map(…)` 会返回一个状态为 rejected 的 promise。

回头看看如何使用：

```js
var p1 = Promise.resolve(21);
var p2 = Promise.resolve(42);
var p3 = Promise.reject('err msg');

Promise.map([p1, p2, p3], function (val, done) {
  Promise.resolve(val)
    .then(
      function (v) { done(v * 2); },
      done
    );
})
  .then(function (vals) {
    console.info(vals); // [42, 84, "err msg"]
  });
```

另外，在并发执行异步任务中，除了 `Promise.all([…])` 之外，也能使用 `async` + `for…of` 函数实现(有点超纲，不过后面也会讲到)：

```js
async function doConcurrent (vals, cb) {
  const results = [];
  const promises = vals.map(val => new Promise((resolve) => cb(val, resolve)));

  for (let promise of promises) {
    try {
      results.push(await promise);
    } catch (e) {
      results.push(e);
    }
  }
  return results;
}
```

```js
doConcurrent([p1, p2, p3], function (val, done) {
  Promise.resolve(val)
    .then(
      function (v) { done(v * 2); },
      done
    );
})
  .then(function (vals) {
    console.info(vals); // [42, 84, "err msg"]
  })
```

## 关于 Promise API 的重要概述(Promise API Recap)
这一节主要是回顾一下之前已经提到的一些 Promise API。

此处作者也提及了关于他写的一个 [《native-promise-only》](http://github.com/getify/native-promise-only) 库的介绍，主要作用是做原生 Promise 的 polyfill。在其书写时候，ES6才刚发布不久，很多浏览器并没有实现这套规范，因此这个库显得十分重要且必要，但是时至今日，Promise 的实现早不是问题，因此其实用性有所下降，但一定也不妨碍我们学习他背后的思想和 code 的手法，这个才是精华。

### new Promise(…) Constructor
构造一个 Promise 的实例，关键字依然是 `new`，同时作为第一个必填的参数(回调函数)，它会被立即执行，并接受两个回调函数，作为确定该 promise 实例的状态，通常我们称其为 `resolve(…)` 和 `reject(…)`：

```js
var p = new Promise(function (resolve, reject) {
  // resolve(…) -> fulfilled or rejected
  // reject(…) -> rejected
});
```

`reject(…)` 没啥好说的，就是把 Promise 的状态确定为 rejected；而 `resolve(…)` 则可能是 fulfilled 的状态或 rejected 的状态，这取决于传进去的参数是什么：
  - 若传递的是 非 Promise、非 thenable 的值，那么一定是 fulfilled 的状态；

  - 若传递的是 Promise 或 thenable 的值，那么就会一直递归去解析，到最后被确定的状态才是 `resolve(…)` 的状态(可能是 fulfilled 也可能是 rejected)；

### Promise.resolve(…) 和 Promise.reject(…)
`Promise.reject(…)` 是创建 rejected 状态的 Promise 的一个简写，比如下面👇的两个形式其实是等价的：

```js
var p1 = new Promise(function (resolve, reject) {
  reject('err msg');
});

var p2 = Promise.reject('err msg');
```

而 `Promise.resolve(…)` 的状态也是不确定的，会根据传入的参数而决定，并且若是传入一个 Promise 的话，它会返回这个 Promise，因此当你不确定值的“类型”时，`Promise.resolve(…)` 是一个没有负担的选择：

```js
var fulfilledTh = {
	then: function(cb) { cb(42); }
};
var rejectedTh = {
	then: function(cb, errCb) {
		errCb('err msg');
	}
};

var p1 = Promise.resolve(fulfilledTh);
var p2 = Promise.resolve(rejectedTh);
```

### then(…) 和 catch(…)
每个 Promise 的实例都有 `then(…)` 和 `catch(…)` 方法，它们用来接收处理 fulfilled 或 rejected 状态的回调函数 —— 当然这都是异步的。

`then(…)` 方法接收两个参数，第一个是用来处理 fulfillment 的回调，第二个则是处理 rejected 的回调，若是其中有一个被省略掉(或传入的是非函数类型的值)，那么就会有一个默认的回调函数来取代比如 `data => data` 或 `function (e) { return e; }`。


`catch(…)` 方法只接受一个参数，同样的也会有默认参数，换句话讲，它其实就相当于 `then(null, …)`：

```js
p.then(fulfilled);

p.then(fulfilled, rejected);

p.catch(rejected); // 等于 p.then(null, rejected);
```

`then(…)` 和 `catch(…)` 都会创建并返回一个新的 Promise，而这也是链式回调的基础。

如果传入它们的回调发生异常，那么返回的 Promise 的状态是 rejected；如果传入它们的回调返回非 Promise 和 非 thenable 的值，那么返回的 Promise 的状态是 fulfilled；而若是传入它们的回调返回了一个 Promise 或 thanble 的值，那会递归解析，直到最终的状态被确定。

### Promise.all([…]) 和 Promise.race([…])
区别

`Promise.all([…])` 和 `Promise.race([…])` 两个静态方法都会创建并返回一个新的 Promise，而这个 Promise 的状态则取决于传入的数组参数：

对 `Promise.all([…])` 而言，只有当你传入的数组中的所有 Promise 的状态都确定为 fulfilled，那么返回的 Promise 才是 fulfilled；而哪怕只有一个的状态确定为 rejected，那返回的 Promise 的状态都被定义为 rejected —— 这种模式被称为 gate —— 所有人都必须在大门开启前到达。

而 `Promise.race([…])` 则是只有第一个确定的 Promise 的状态会作为返回的 Promise 的状态 —— 这样的模式被称为 latch —— 只有第一个抵达门闩的人才能够进入。

```js
const pArr = [
  Promise.resolve(42),
  Promise.resolve('Hello World'),
  Promise.reject('err msg')
];

Promise.race(pArr)
  .then(function(msg){
    console.log(msg);		// 42
  });

Promise.all(pArr)
  .catch(function(err){
    console.error(err);	// "err msg"
  });

Promise.all(pArr.slice(0, 2))
  .then(function(msgs){
    console.log(msgs);	// [42, "Hello World"]
  });
```

**Warning**：若是传入一个空数组作为参数时，`Promise.all([…])` 和 `Promise.race([…])` 表现的行为有差异：前者会直接返回一个 fulfilled 状态的 Promise，而后者则会永远的 pending 在那里！

原生的 Promise 作为一个从回调地狱过度到新版的 JS 异步编程的良好开端是十分靠谱的，但它依然在某些情况下不能够很好的完全满足异步编程的需求，这也为一些基于 Promise 思想的库提供的驱动力。

## Promise 的限制(Promise Limitations)
虽然之前的内容中已经涵盖了这部分的一些内容，但是依然很有必要单独的将其罗列出来。

### 异常处理的顺序(Sequence Error Handling)
Promise 的链式调用仅仅是将一堆 Promise 组合起来的一种形式，因此你没办法仅通过一个实体，就能建立对 Promise 链的全部引用；而这也也意味着说，没有其他的办法能够监控到异常的发生。

一旦 Promise 链中有错误发生，它就会无限的向下传播，直到遇到一个注册的异常处理方法 —— 在这种模式下

```js
const p = foo(42)
  .then(STEP2)
  .then(STEP3);
```

`p` 的指向的 Promise 并不是第一个 `foo(42)`，而是最后一个，即 `then(STEP3)` 的返回值，那即意味着你可以自行注册一个异常处理的方法：

```js
p.catch(handleErrors);
```

通常情况下👆都能很好的处理异常的问题，不过，若是在之前的其他步骤注册了异常处理的方法，这就缺失了一个通知 `handleErrors` 有异常已经被处理的机制。

但这也不是只此一家 —— `try…catch` 同样也有类似的问题，`catch` 会将异常情况吞掉而没有任何形式的通知。

由于不能对中间步骤的 Promise 保留引用，因此也没办法写出一个能够全面覆盖 Promise 异常情况的处理函数。

### 单一的值(Single Value)
根据定义，Promise 只能有一个单一的 fulfillment 或 rejection 的值。通常的解决办法是将多个值包裹在一个对象或数组中，而后再解析出来 —— 不仅很尴尬，而且很冗余。

#### 值的分割(Splitting Values)
有时候这也是一个让你将问题分解成多个 Promise 来处理的信号，就比如你有一个异步返回两个值的 `foo(…)` 方法：

```js
function getY(x) {
	return new Promise(function (resolve, reject){
		setTimeout(function (){
			resolve((3 * x) - 1);
		}, 100);
	});
}

function foo(bar,baz) {
	const x = bar * baz;

	return getY(x)
    .then(function (y){
      return [ x,y ];
    });
}

foo(10, 20)
  .then(function (msgs){
    const x = msgs[0];
    const y = msgs[1];

    console.log(x, y);  // 200 599
  });
```

借助 `Promise.all([…])` 我们将👆

```js
function foo(bar,baz) {
	const x = bar * baz;

	return [
		Promise.resolve(x),
		getY(x)
	];
}

Promise.all(
	foo( 10, 20 )
)
  .then(function (msgs){
    const x = msgs[0];
    const y = msgs[1];

    console.log(x, y);
  });
```

但由 Promise 组成的数组就一定优于将一组值作为数组在一个 Promise 中返回吗？就语法而言，好像并没有！

但这样的方式和 Promise 的实际思路的确更为贴切，也更容易组织和管理代码。不过显然解决的办法不止一种。

#### 参数的扩展运算(Unwrap/Spread Arguments)
有没有更讨巧的方式呢？社区中的大神随手一挥 —— `apply`：

```js
function spread(fn) {
	return Function.apply.bind(fn, null);
}

Promise.all(
	foo(10, 20)
)
  .then(
    spread(function (x, y){
      console.log(x, y);
    })
  );
```

ES6 显然也提供了另一种优雅的方式：

```js
Promise.all(
	foo(10, 20)
)
  .then(
    function (data) {
      const [x, y] = data;
      console.log(x, y);
    }
  );
```

但最好的莫过于 ES6 提供的对于函数参数的数组解构了：

```js
Promise.all(
	foo(10, 20)
)
  .then(
    function ([x, y]) {
      console.log(x, y);
    }
  );
```

`one-value-per-Promise` 的特性的确恶心，不过好在也有解决方案。

### 没有回头路(Single Resolution)
众所周知，Promise 的状态一旦确定，就没办法更变，要么 fulfillment，要么 rejection —— 大多数的情况这是好事，但依然有局限，就比如 Promise 无法适应处理事件或者流的这种需要多次解析的情况，如按钮的点击事件：

```js
const p = new Promise(function (resolve,reject){
  // `click(…)` 绑定了 DOM 上的 click 事件
	click('#btn', resolve);
});

p
  .then(function (evt){
    const btnID = evt.currentTarget.id;
    return request('http://some.url.1/?id=' + btnID);
  })
  .then(function (text){
    console.log(text);
  });
```

👆这样的事件处理只用使用一次，因为下次在点击事件触发的时候，`p` 的状态早已经被确定了。

为了解决这个问题，你可能要转化一下范式 —— 每次事件被触发的时候都创建一个新的 Promise 链：

```js
click('#btn', function(evt) {
	const btnID = evt.currentTarget.id;

	requestrequest('http://some.url.1/?id=' + btnID)
    .then(function (text){
      console.log(text);
    });
});
```

👆先不管这样写的代码有多丑，这样的代码违反了 职责任分离(separation of concerns/capabilities SoC) 的原则 —— 就比如你想把事件注册和事件处理的代码逻辑分开，而这样的模式在没有其他 helper 方法的帮助下，显然没办法做到。

**注意**：社区中也有很多对这样的限制的解决方案，就比如 [RxJS](https://archive.codeplex.com/?p=rxjs)。但问题在于这种解决方案一个是太重了，另一个是这样的第三方库是否遵循了 Promise 的 *trustable* 原则还未可知。

### 惯性(Inertia)
**A code base in motion (with callbacks) will remain in motion (with callbacks) unless acted upon by a smart, Promises-aware developer.** —— **想要打破现状总是很难的**

Promise 不是一个一刀切(one-size-fits-all)的方案，想要将原来的异步回调函数的代码全部重构成 Promise 的代码，显然工作量很大，而且还容易出幺蛾子。

但这并不意味着就不用 Promise 了，相反，社区中有很多开源的项目能够支持将代码 Promise化(Promisory)

### 不可取消的 Promise(Promise Uncancelable)
一旦你创建了一个 Promise，并且注册了处理方法在其中，就没办法取消了。

**注意**：有很多的第三方库提供了取消 Promise 的工具，甚至有的开发者希望原生支持取消 Promise，但这其实是弊大于利的 —— 一旦你能够影响到 Promise 的结果，那么它就不在是 trustable 了，那就等于又回到了过去的回调函数的噩梦中去了。

通常情况下，要实现 Promise 的 cancel 功能，只能通过在更高的抽象层次来解决：

```js
let OK = true;

const p = foo(42);

Promise.race([
	p,
	timeoutPromise(3000)
    .catch( function(err){
      OK = false;
      throw err;
    })
])
  .then(
    doSomething,
    handleError
  );

p
  .then( function(){
    if (OK) {
      // 只有未超时才会触发
    }
  });
```

### Promise 的性能(Promise Performance)
Promise 肯定会比相同代码的 回调函数 慢一丢丢 —— 加了一层中间层不慢怎么可能。但这样的性能比较真的没太多意思，最起码你也应该用 带有处理信任问题的回调函数 和 Promise 比较才有意义不是么？

而问题的核心不在于比较性能的优劣，而在于若是你真的差那么一丢丢的性能才能解决问题的话，那你为何不用效率更高的语言(比如 `C++`)来代替JS，从而实现你的需求呢？

况且，就算是慢了一些，但 Promise 带来的优势，比如可信任、链式调用等，从任何角度看都是一笔划算的买卖。即便是有诸多的限制，但兴许是你不理解 Promise 带来的好处才会考虑的吧？！

## 回顾(Review)
Promise 主要解决的问题是回调函数的控制反转，从而导致信任缺失的问题。它并没有摆脱回调函数，它只是在我们的代码和第三方库之间作为一个可信任的中间者，帮助我们解决问题。

Promise 的链式调用当然也部分优化了异步代码的控制流，但显然还不够。