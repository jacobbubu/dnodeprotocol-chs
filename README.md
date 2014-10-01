概述
====

[dnode](https://github.com/substack/dnode) 是一个非常灵巧的异步 RPC 系统，由 [substack](https://github.com/substack) 在两年前用 [node.js](http://nodejs.org) 开发。本身 dnode 既可以运行在 node.js 中，也可以通过 [Browserify](https://github.com/substack/node-browserify) 运行在浏览器中。

dnode 是传输协议无关的，尤其在 node.js 中，任何支持 Stream 的传输协议都可以应用 dnode。除了 Javascript，dnode 也有 Perl，PHP，Ruby，Objective-C 和 Java 等不同语言的实现。

dnode 吸收了 Javascript 动态语言的灵活性和 node.js 的异步特性，即简单轻巧，又适用于大部分的“现代” RPC 场景。

原本我只是想翻译 [dnode-protocol](https://github.com/substack/dnode-protocol/blob/master/doc/protocol.markdown))，后来发现按照英文直译其实很难理解，因此扩展为一个小教程。

如果你之前没用过 dnode，那么在阅读本文之前，建议先阅读 [dnode](https://github.com/substack/dnode)。

协议文档
========

dnode 的消息采用 JSON 方式表示， 以新行符进行消息分割。和强类型的 RPC 协议不同，dnode 并不需要通信的两端实现交换 IDL (Interface Definition Language)，例如：CORBA 或者 COM，也不对实际的方法签名进行检查。就像 Javascript 本身，一切都在执行时才真的揭晓。

dnode 的消息有两种类型，“方法交换”消息和“方法调用”消息。

方法交换
--------

在通信连接建立之后，RPC 的两端首先要做的就是和对方交换自己的方法信息，这样在本地就可以生成一个远端的代理对象。这是通过以下消息实现的：

``` json
{
    "method" : "methods",
    "arguments" : [ { "timesTen" : "[Function]", "moo" : "[Function]" } ],
    "callbacks" : { "0" : ["0","timesTen"], "1" : ["0","moo"] }
}
```

该 JSON 消息为了阅读方面，并没有写到一行里。

对于“方法交换”消息，`method` 字段的值约定为 "methods"（所以你不应该把自己的 RPC 方法命名为 "methods"）。该消息的 `arguments` 字段是仅包含“一个”元素的数组，该元素定义了本方所拥有的方法名称。其中的 "[Function]" 仅仅是一个占位符，表示这是一个方法(函数)。

`callbacks` 则是用来为 `arguments` 中的方法名称设定对应的整数 ID。例如，`arguments` 中的方法 "timesTen"，其 ID 在 `callbacks` 中被定义为 "0"，而 "moo" 的 ID 则为 "1"。 `"0" : ["0","timesTen"]` 的含义为：ID "0"，指代的是 `arguments` 数组的第 0 个元素，且该元素（元素为一个 Object）的 key 为 "timesTen" 的值。`["0","timesTen"]` 在这里称为指向 `arguments` 内容的路径。类似地，`["0","moo"]` 是 ID "1" 的 `arguments` 中内容的路径。因此可以看到，某个整数 ID 可以实际代表 `arguments` 数组中任意深度的某个具体值。

使用 ID，一方面是为了今后调用时可以用 ID 替代方法名称以节约流量消耗，另一方面，也是为了应对异步回调时的匿名函数的情况。继续阅读“方法调用”章节即可有更深入的理解。

方法调用
--------

我们看一个例子，其中既包含了“方法交换”也包含了“方法调用”的过程：

``` js
var proto = require('dnode-protocol');

var s = proto({
    x : function (f, g) {
        setTimeout(function () { f(5) }, 200);
        setTimeout(function () { g(6) }, 400);
    },
    y : 555
});
var c = proto();

s.on('request', c.handle.bind(c));
c.on('request', s.handle.bind(s));

c.on('remote', function (remote) {
    function f (x) { console.log('f(' + x + ')') }
    function g (x) { console.log('g(' + x + ')') }
    remote.x(f, g);
});

s.start();
c.start();
```

这是 module [`dnode-protocol`](https://github.com/substack/dnode-protocol.git) 中的例子。这个 module 是 dnode 协议的 Javascript 实现，其本身和通信协议是无关的。具体 [`dnode-protocol`](https://github.com/substack/dnode-protocol.git) 的 API 的文档请直接去它的 GitHub 站点。

上例中 `c` 和 `s` 是通信的两端，你可以理解为 Client 和 Server。`s` 声明了 `x` 方法和 `y` 常量(`s = proto({x: function{...}, y: 555})`)，而 `c` 没有声明任何方法(`c = proto()`)。
一旦 `s` 或者 `c` 启动(`s.start();c.start()`)，它们就会主动生成 “方法交换”的消息。该消息通过发布 `request` 事件发送出来。上例中的 `s.on('request', c.handle.bind(c));` 表示，一旦 `s.start()`，那么就将发出包含“方法交换”消息 `request` 事件(具体消息对象是 `on('request', function (req, argv) {}` 中的 req)，然后由 `c` 来处理，反之亦然。

`c` 收到这个消息之后，会转变为自身的 `remote` 事件(`c.on('remote', function (remote) {}`)，回调函数中的参数 `remote` 实际上是 `s` 的一个方法代理，执行 `remote.x(...)` 或者 `remote.y()` 将最终调用到 `s` 的 `x` 和 `y` 方法。

上例中 `c.on('remote', function (remote) {}` 最终调用了 `remote.x(f, g)`, 意味着为 `s.x` 方法传递了 `f` 和 `g` 两个回调函数，因此最终对 `s.x` 的调用又会回到 `c` 中的函数 `f` 和 `g`。该例最终的执行结果为：

```
f(5)
g(6)
```

`f(5)` 先返回，因为在 `s` 上对 `f` 的回调是收到消息后 200 毫秒，而 `g` 则为 400 毫秒。你可以自己调整这些值来实验一下。

通信的一段无论是否有 RPC 方法，都要向对方发送“方法交换”消息，例如，`c` 发出的“方法交换”消息为

``` json
{
    "method": "methods",
    "arguments": [ {} ],
    "callbacks": {},
    "links": []
}
```

可见都是空的，即使如此，`s` 上仍然也会生成一个代表 `c` 的 `remote` 对象，只是其中一个方法都没有。

而 `s` 产生的“方法交换”消息的内容为：

``` json
{
    "method": "methods",
    "arguments": [ { "x": "[Function]", "y": 555 } ],
    "callbacks": { "0": [ "0", "x" ] },
    "links": []
}
}
```

我们可以验证一下之前对“方法交换”消息的说明。`method` 固定取值为 "methods"。`arguments`是个数组，其第一个元素是个对象，其两个 keys `x` 和 `y` 分别对应一个方法和一个常量(`555`)。
`callbacks` 声明了一个方法 ID 0，指向 `arguments` 的第 0 个元素的 "x" 属性的值。

“方法交换”完成后，`c` 会得到 `s` 的代理对象 `remote`，随后通过 `remote.x(f, g)` 调用 `s.x`，其对应的消息如下：

``` json
{
    "method": 0,
    "arguments": [ "[Function]", "[Function]" ],
    "callbacks": { "0": ["0"], "1": ["1"] },
    "links": []
}
```

其中 `method` 的值就是上一个消息中的 `callbacks` 中声明的 ID，对应到 `s.x`。在这里，`"method": 0` 或者 `"method": "x"` 都是合法的，只是用 ID 在大多数情况下更精简一些。而 `arguments` 中的两个元素 `"[Function]"` 则和之前的消息不同，并没有对应的名称。这是因为 `remote.x(f, g)` 中的 `f` 和 `g` 本就是匿名函数，不会出现在 `s` 那里对应 `c` 的 `remote` 代理对象中。而 `callbacks` 中则分别为这两个匿名函数准备好了 ID "0" 和 "1"。最终，`s.x` 执行时回调 `f` 和 `g` 就是通过这两个 ID 完成的，消息如下：

对应 `setTimeout(function () { f(5) }, 200)` 的消息:

``` json
{
    "method": 0,
    "arguments": [5],
    "callbacks": {},
    "links": []
}
```

对应 `setTimeout(function () { g(6) }, 400)` 的消息:

``` json
{
    "method": 1,
    "arguments": [6],
    "callbacks": {},
    "links": []
}
```

所有的 dnode 消息超不出以下四个字段：

* method :: String 或 Integer
* arguments :: Array
* callbacks :: Object
* links :: Array

`links` 请见下节。

links
-----

"links" 是一个可选的字段，用来表示 arguments 中循环引用的数据结构。

我们看一段代码（其中 `fn` 是一个远程方法）：

``` js
    var data = { a : 5, b : [ { c : 5 } ] };
    data.b.push(data);
    fn(data);
```

`data.b` 的第二个元素又指回了 `data`，因此 `data` 是一个循环引用数据结构。

其产生的 message 为：

``` json
{
    "method" : 12,
    "arguments" : [ { "a" : 5, "b" : [ { "c" : 5 } ] } ],
    "callbacks" : {},
    "links" : [ { "from" : [ 0 ], "to" : [ 0, "b", 1 ] } ]
}
```

`links` 是个数组，每个元素为一个包含 `from` 和 `to` 的 link 定义。上例中的唯一一个 link 定义表示：arguments 中的第一个元素的 key 为 "b" 的对象的第二个元素（`"to" : [ 0, "b", 1 ] }`），其值指向 arguments 的第一个元素（`"from" : [ 0 ]`）。

请注意，link 也不仅仅是用来表示循环引用的数据结构，引用本身也可以用来消除重复的数据。