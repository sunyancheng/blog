# 浅析 Web Workers 及 应用

## 前言

在浏览器中，由于 JavaScript 引擎与 GUI 渲染线程是互斥的，所以当我们在 JavaScript 中执行一些计算密集型或高延迟的任务的时候，会导致页面渲染被阻塞或拖慢。为了解决这个问题，提高用户体验，HTML5 为我们带来了 Web Workers 这一标准。

## 概述

作为 HTML5 标准中的一部分，`Web Workers` 定义了一套 API，允许一段 JavaScript 程序运行在主线程之外的 Worker 线程中。 在主线程运行的同时，Worker 线程可以在后台独立运行，处理一些计算密集型或高延迟的任务，等 Worker 线程完成计算任务，再把结果返回给主线程。从而保证主线程（通常是 UI 线程）不会因此被阻塞或拖慢。

常见的 Web Workers 主要有以下三种类型：

- Dedicated Workers
- Shared Workers
- Service Workers

## Dedicated Workers

即`专用 Workers`， 仅能被生成它的脚本所使用，且只能与一个页面渲染进程进行绑定和通信, 不能多 Tab 共享。浏览器的支持情况如下图：
![https://s1.firstleap.cn/s/104395/083621334062269571618899799914.png](https://s1.firstleap.cn/s/104395/083621334062269571618899799914.png)

## 专用 Workers 的基本用法

### 1. 创建 worker 线程方法：

我们在主线程 JS 中调用 new 命令，然后实列化 Worker()构造函数，就可以创建一个 Worker 线程了，代码如下所示：

```js
var worker = new Worker("work.js");
```

`Worker()` 构造函数的参数是一个脚本文件，该文件就是 Worker 线程需要执行的任务，需要注意的是，由于 Web Workers 有同源限制，因此这个脚本必须从网络或者本地服务器读取。

### 2. 主进程发送数据

接下来，我们就可以从主线程向子线程发送消息了，使用 `worker.postMessage()`方法，向 Worker 发送消息。代码如下所示：

```js
worker.postMessage("Hello LeapFE");
```

`worker.postMessage` 方法可以接受任何类型的参数，甚至包括二进制数据。

### 3. Worker 监听函数

Worker 线程内部需要有一个监听函数，监听主线程/其他子线程 发送过来的消息。监听事件为 `message`. 代码如下所示：

```js
addEventListener('message', function(e) { postMessage('子线程向主线程发送消息: ' + e.data); close(); // 关闭自身 });`
```

子线程接收到主进程发来的数据，然后执行相应的操作，最后把结果再返回给主线程，

### 4.主进程接收数据

主线程通过 `worker.onmessage` 指定监听函数，接收子线程传送回来的消息，代码如下所示：

```js
worker.onmessage = function (event) {
  console.log("接收到的消息为: " + event.data);
};
```

从事件对象的 data 属性中可以获取到 Worker 发送回来的消息。

如果我们的 Worker 线程任务完成后，我们的主线程需要把它关闭掉，代码如下所示：

```js
worker.terminate();
```

### 5. importScripts() 方法

Worker 内部如果需要加载其他的脚本的话，我们可以使用 `importScripts()` 方法。代码如下所示：

```js
importScripts("a.js");
```

如果要加载多个脚本的话，代码可以写成这样：

```js
importScripts('a.js', 'b.js', 'c.js', ....);
```

### 6. 错误监听

主线程可以监听 Worker 线程是否发生错误，如果发生错误，Worker 线程会触发主线程的 `error` 事件。

```js
worker.onerror = function (e) {
  console.log(e);
};
```

如果是在 Worker 中如果发生错误的话， 可以通过`throw new Error()` 将错误暴露出来，但这个错误无法被主线程获取，只能在 Worker 的 `console` 中看到“错误未捕获提示”的错误提示，而不是主线程的 `console`！

```js
// worker.js内部：

// ... other code
throw new Error("test error");
```

## Shared Workers

即`共享 Workers`， 可以看作是专用 Workers 的拓展，除了支持专用 Workers 的功能之外，还可以被不同的 window 页面，iframe，以及 Worker 访问（当然要遵循同源限制），从而进行异步通信。浏览器的支持情况如下图：
![https://s1.firstleap.cn/s/104395/92874481873100351618899810253.png](https://s1.firstleap.cn/s/104395/92874481873100351618899810253.png)

## 共享 Workers 的基本用法

### 1. 共享 Workers 的创建

创建共享 Workers 可以通过使用 SharedWorker() 构造函数来实现，这个构造函数使用 URL 作为第一个参数，即是指向 JavaScript 资源文件的 URL。代码如下所示：

```js
var worker = new SharedWorker("sharedworker.js");
```

### 2. 共享 Workers 与主进程通信

共享 Workers 与主线程交互的步骤和专用 Worker 基本一样，只是多了一个 port：

```js
// 主线程：
const worker = new SharedWorker("worker.js");
const key = Math.random().toString(32).slice(-6);
worker.port.postMessage(key);
worker.port.onmessage = (e) => {
  console.log(e.data);
};
```

```js
// worker.js：
const buffer = [];
onconnect = function (evt) {
  const port = evt.ports[0];
  port.onmessage = (m) => {
    buffer.push(m.data);
    port.postMessage("worker receive:" + m.data);
  };
};
```

在上面的代码中，需要注意的地方有两点：

1. `onconnect` 当其他线程创建 sharedWorker 其实是向 sharedWorker 发了一个链接，worker 会收到一个 connect 事件
2. `evt.ports[0]` connect 事件的句柄中 evt.ports[0]是非常重要的对象 port，用来向对应线程发送消息和接收对应线程的消息

## Service workers

在目前阶段，Service Worker 的主要能力集中在网络代理和离线缓存上。具体的实现上，可以理解为 Service Worker 是一个能在网页关闭时仍然运行的 Web Worker。浏览器的支持情况如下图：
![https://s1.firstleap.cn/s/104395/330800979133963671618899806918.png](https://s1.firstleap.cn/s/104395/330800979133963671618899806918.png)

PS: `Service Workers`涉及的功能点比较多，因篇幅有限，本文将暂不进行介绍，我们会在后面的更新中再详细解析。

## 专用 Worker 和 共享 Worker 的应用场景

如上文已经提到的，Worker 可以在后台独立运行，不阻塞主进程，最常见的使用 Worker 的场景就是处理一些计算密集型或高延迟的任务。

### 场景一: 使用 专用 Worker 来解决耗时较长的问题

我们在页面中有一个 input 输入框，用户需要在该输入框中输入数字，然后点击旁边的计算按钮，在后台计算从 1 到给定数值的总和。如果我们不使用 Web Workers 来解决该问题的话，如下 demo 代码所示：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Web Worker</title>
  </head>
  <body>
    <h1>从1到给定数值的求和</h1>
    输入数值: <input type="text" id="num" />
    <button onclick="calculate()">计算</button>

    <script type="text/javascript">
      function calculate() {
        var num = parseInt(document.getElementById("num").value, 10);
        var result = 0;
        // 循环计算求和
        for (var i = 0; i <= num; i++) {
          result += i;
        }
        alert("总和为：" + result + "。");
      }
    </script>
  </body>
</html>
```

如上代码，然后我们输入 1 百亿，然后让计算机去帮我们计算，计算的时间应该要 20 秒左右的时间，但是在这 20 秒之前的时间，那么我们的页面就处于卡顿的状态，也就是说什么都不能做，等计算结果出来后，我们就会看到如下弹窗提示结果了，如下所示：

那现在我们尝试使用 Web Workers 来解决该问题，把这些耗时操作使用 Worker 去解决，那么主线程就不影响页面假死的状态了，我们首先把 index.html 代码改成如下：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Web Worker</title>
  </head>
  <body>
    <h1>从1到给定数值的求和</h1>
    输入数值: <input type="text" id="num" />
    <button id="calculate">计算</button>
    <script type="module">
      // 创建 worker 实列
      var worker = new Worker("./worker1.js");

      var calDOM = document.getElementById("calculate");
      calDOM.addEventListener("click", calculate);

      function calculate() {
        var num = parseInt(document.getElementById("num").value, 10);
        // 将我们的数据传递给 worker 线程，让我们的 worker 线程去帮我们做这件事
        worker.postMessage(num);
      }

      // 监听 worker 线程的结果
      worker.onmessage = function (e) {
        alert("总和值为:" + e.data);
      };
    </script>
  </body>
</html>
```

如上代码我们运行下可以看到，我们点击下计算按钮后，我们使用主线程把该复杂的耗时操作给子线程处理后，我们点击按钮后，我们的页面就可以操作了，因为主线程和 Worker 线程是两个不同的环境，Worker 线程的不会影响主线程的。因此如果我们需要处理一些耗时操作的话，我们可以使用 Web Workers 线程去处理该问题。

### 场景二: 使用 共享 Worker 实现跨页面数据共享

下面我们给出一个例子：创建一个共享 Worker 共享多个 Tab 页的数据，实现一个简单的网页聊天室的功能。

![https://s1.firstleap.cn/s/104395/33920652922595031618927790538.png](https://s1.firstleap.cn/s/104395/33920652922595031618927790538.png)

首先在 `index.html` 中设计简单的聊天对话框样式， 同时引入 main.js：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Shared Worker Example</title>
    <style>
      ul li {
        float: left;
        list-style: none;
        margin-top: 10px;
        width: 100%;
      }

      ul li > span {
        font-size: 10px;
        transform: scale(0.8);
        display: block;
        width: 17%;
      }

      ul li > p {
        background: rgb(140, 222, 247);
        border-radius: 4px;
        padding: 4px;
        margin: 0;
        display: inline-block;
      }

      ul li.right {
        float: right;
        text-align: right;
      }

      ul li.right > p {
        background: rgb(132, 226, 140);
      }

      ul li.right > span {
        width: 110%;
      }

      #chatList {
        width: 300px;
        background: #fff;
        height: 400px;
        padding: 10px;
        border: 4px solid #de8888;
        border-radius: 10px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <section>
        <p id="user"></p>
        <ul id="chatList" style="width: 300px"></ul>
        <input id="input" />
        <button id="submitBtn">提交</button>
      </section>
    </div>
    <script src="./main.js"></script>
  </body>
</html>
```

在 main.js 中，我们初始化一个 `SharedWorker` 实例

```js
window.onload = () => {
  const worker = new SharedWorker("./shared-worker.js");
  const chatList = document.querySelector("#chatList");

  let id = null;

  worker.port.onmessage = (event) => {
    const { data } = event;
    switch (data.action) {
      case "id": // 接收 Worker 实例化成功之后返回的 id
        id = data.value;
        document.querySelector("#user").innerHTML = `Client ${id}`;
        break;

      case "message": // 接收 Worker 返回的来自各个页面的信息
        chatList.innerHTML += `<li class="${
          data.id === id ? "right" : "left"
        }"><span>Client ${data.id}</span><p>${data.value}</p></li>`;
        break;

      default:
        break;
    }
  };

  document.querySelector("#submitBtn").addEventListener("click", () => {
    const value = document.querySelector("#input").value;
    // 将当前用户 ID 及消息发送给 Worker
    worker.port.postMessage({
      action: "message",
      value: value,
      id,
    });
  });
};
```

在 `shared-worker.js` 接收与各页面的连接，同时转发页面发送过来的消息

```js
const connectedClients = new Set();
let connectID = 1;

function sendMessageToClients(payload) {
  //将消息分发给各个页面
  connectedClients.forEach(({ client }) => {
    client.postMessage(payload);
  });
}

function setupClient(clientPort) {
  //通过 onmessage 监听来自主进程的消息
  clientPort.onmessage = (event) => {
    const { id, value } = event.data;
    sendMessageToClients({
      action: "message",
      value: value,
      id: connectID,
    });
  };
}

// 通过 onconnect 函数监听，来自不同页面的 Worker 连接
onconnect = (event) => {
  const newClient = event.ports[0];
  // 保存连接到 Worker 的页面引用
  connectedClients.add({
    client: newClient,
    id: connectID,
  });

  setupClient(newClient);

  // 页面同 Worker 连接成功后, 将当前连接的 ID 返回给页面
  newClient.postMessage({
    action: "id",
    value: connectID,
  });
  connectID++;
};
```

在上面的共享线程例子中，在主页面即各个用户连接页面构造出一个共享线程对象，然后通过 `worker.port.postMessage` 向共享线程发送用户输入的信息。同时，在共享线程的实现代码片段中定义 `connectID`， 用来记录连接到这个共享线程的总数。之后，用 `onconnect` 事件处理器接收来自不同用户的连接，解析它们传递过来的信息。最后，定义一个了方法 `sendMessageToClients` 将消息分发给各个用户。

## 总结

Web Workers 确实给我们提供了优化 Web 应用的新可能，通过使用 Web Workers 来合理地调度 JavaScript 运行逻辑，可以在面对无法预测的低端设备和长任务时，保证 GUI 依旧是可响应的。

或许未来，使用 Web Workers 进行编程或许会成为新一代 Web 应用开发的标配或是最佳实践。
