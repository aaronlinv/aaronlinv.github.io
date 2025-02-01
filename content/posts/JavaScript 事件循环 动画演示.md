---
date: '2024-04-17T07:49:33+08:00'
title: 'JavaScript 事件循环 动画演示'
categories: ["前端"]
---

在前端代码中很经常看到使用 `setTimeout(fn, 0)`，如下面代码所示，乍一看很多余，但是移除了可能会出现一些奇奇怪怪的问题。要解释这个就需要理解 **事件循环（Event Loop）**，下面会通过一些例子和动画来辅助理解事件循环

```js
setTimeout(() => {
  // 调用一些方法
}, 0)
```

## 为什么使用事件循环

JS 是单线程的（[浏览器和 Node则是多线程的](https://www.reddit.com/r/node/comments/nqxelw/why_everyone_is_saying_that_js_is_single_threaded/)），为了避免 **渲染主线程** 阻塞，需要异步，**事件循环** 是异步的实现方式

浏览器在一个渲染主线程中运行一个页面中的所有 JavaScript 脚本，以及呈现布局，回流，和垃圾回收。为了避免 **同步** 的执行方式导致渲染主线程阻塞，使得页面卡死，所以浏览器采用异步的方式：渲染主线程将任务交给其他线程去处理，自身 **立即结束** 任务的执行，转而执行后续代码，当其他线程完成时，将事先传递的回调函数包装成任务，加入到对应的消息队列的**末尾**排队，等待渲染主线程调度执行

流程：
1. 渲染主线程执行全局 JS，需要异步的任务放到对应的队列，如果是 `setTimeout` 则会有线程计时，到了指定时间会将任务放入 `延时队列`（并非立即执行）
2. 渲染主线程为空时，按队列的优先级依次选择队列（最先执行微队列的任务），依次按顺序执行各个队列的任务

任务没有优先级，而消息队列有优先级，不同任务分属于不同队列：[参考 W3C 规范](https://html.spec.whatwg.org/multipage/webappapis.html#generic-task-sources)。[微队列优先级最高](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)，接着是交互队列然后才是延时队列

常见队列：

- 微队列（microtask）：⽤户存放需要最快执⾏的任务，优先级「最⾼」，通过 `Promise.resolve().then()` ⽴即把⼀个函数添加到微队列
- 交互队列：⽤于存放⽤户操作后产⽣的事件处理任务，优先级「⾼」
- 延时队列：⽤于存放计时器到达后的回调任务，优先级「中」



## 事件循环

下面例子来自于：[《WEB前端大师课》](https://ke.qq.com/course/5892689)，大块的文字描述相对没那么直观，所以用 Keynote 做了 gif 方便理解（如果有更好的做 gif 的方式可以留言告诉我）

### 1. JS阻碍页面渲染

JS 修改了 DOM 后，并不会马上显示在页面上，需要进行 **绘制** 后才会显示页面变更

```html
<!DOCTYPE html>
<html lang="en">
  <head></head>
  <body>
    <h1>初始h1</h1>
    <button>change</button>
    <script>
      var h1 = document.querySelector('h1');
      var btn = document.querySelector('button');

      function delay(duration) {
        var start = Date.now();
        while (Date.now() - start < duration) {}
      }
      
      btn.onclick = function () {
        h1.textContent = '修改h1 textContent';
        delay(3000);
      };
    </script>
  </body>
</html>
```

![01绘制任务](../JavaScript事件循环动画演示/1929786-20240416173932693-109077340.gif)

效果：点击 `change` 后，页面卡死，3s 后 `h1` 内容变更为：`修改h1 textContent`

### 2. 延迟队列

`setTimeout` 到达指定时间可能并不会立即执行

```js
setTimeout(function () {
  console.log(1);
}, 0);

function delay(duration) {
  var start = Date.now();
  while (Date.now() - start < duration) {}
}

delay(3000);

console.log(2);
```

![02settimeout](../JavaScript事件循环动画演示/1929786-20240416174000096-2033848170.gif)

效果：卡死 3s 后输出 2 1

### 3. 微队列

使用 `Promise.resolve().then` 可以将任务直接添加到微队列

```js
setTimeout(function () {
  console.log(1);
}, 0);

Promise.resolve().then(function () {
  console.log(2);
});

console.log(3);
```

![03微队列](../JavaScript事件循环动画演示/1929786-20240416174033217-629343889.gif)

效果：依次输出 3 2 1

### 4. 复杂情况

```js
function a() {
  console.log(1);
  Promise.resolve().then(function () {
    console.log(2);
  });
}
setTimeout(function () {
  console.log(3);
  Promise.resolve().then(a);
}, 0);

Promise.resolve().then(function () {
  console.log(4);
});

console.log(5);
```

![04复杂Promise](../JavaScript事件循环动画演示/1929786-20240416174055454-1565610014.gif)

效果：依次输出 5 4 3 1 2

### 拓展

理解了上面的概念，可以尝试分析一下 [现代 JavaScript 教程 事件循环例子](https://zh.javascript.info/event-loop#xia-fang-zhe-duan-dai-ma-de-shu-chu-shi-shi-mo)，检查一下是否理解了事件循环

## 参考资料

[2024 年我还在写这样的代码](https://twitter.com/randyloop/status/1779145003824288010)
[为什么 JS 要加入 setTimeout， css 的 transition 才能生效](https://v2ex.com/t/633069)
[深入理解和使用 Javascript 中的 setTimeout(fn,0)](https://yugasun.com/post/understand-settimeout-fn-0)
[主线程](https://developer.mozilla.org/zh-CN/docs/Glossary/Main_thread)
[并发模型与事件循环](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Event_loop)
[异步 JavaScript](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Asynchronous)
[调度：setTimeout 和 setInterval](https://zh.javascript.info/settimeout-setinterval)