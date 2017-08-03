---
title: 写一个 JavaScript 框架（三）
date: 2017-08-02 11：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是写一个 JavaScript 框架系列文章的第三篇，这次我会阐明几种在浏览器中求值的方式以及它们将引起的一些问题。我还会介绍一个依赖于一些新的鲜为人知的 JavaScript 特性的方法。

这个系列包括以下几个章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. 沙箱求值（当前章节）
4. 数据绑定简介
5. 用 ES6 Proxy 实现数据绑定
6. 自定义元素
7. 客户端路由

<!-- more -->

## 恶魔般的 eval
> eval() 函数将传入的字符串当做 JavaScript 代码执行。

一个常见的执行代码的方案是使用 eval() 函数。通过 eval() 执行的代码能够访问到全局和闭包的作用域，但这会引起代码注入的安全问题，这也是让它成为 JavaScript 特性中最臭名昭著的一个的原因。
尽管 eval() 让人很厌恶，但是在某些场景下不得不说它十分有用。绝大多数的现代前端框架都需要它的功能，但是由于上述提及的问题而不敢使用它。结果就是，许多在沙箱中求值而不是全局作用域下的替代方案如雨后春笋般涌现。沙箱可以阻止代码访问我们的安全数据，通常它只是一个普通的 JavaScript 对象取代了原来的全局对象，用来运行代码。

## 常用方案
最常用的 eval() 的替代方案是用一个分步过程完全重写，它由两个步骤组成 - 分析和编译传递过来的字符串。首先解析器会创建一个抽象的语法树，然后由编译器遍历这个树，在沙箱中编译成正常的代码。
这是一个广泛应用了的方案，但是被认为对于如此简单的一件事而言，它过于庞大和复杂了。比起修复 `eval()`，重写所有代码本身就有可能导致大量的 bug，并且还需要频繁更新代码以遵循编程语言的更新。

## 替代方案
[NX](http://nx-framework.com/) 避免重写所有原生代码。通过一个迷你的库执行代码，这个库使用了一些新的并且鲜为人知的 JavaScript 特性。
这一部分将会逐步介绍一些 JavaScript 特性，并且使用它们来剖析[nx-compile](https://github.com/RisingStack/nx-compile)这个执行代码的库。这个库有一个叫 `compileCode()` 函数，工作方式如下：

```JavaScript
const code = compileCode('return num1 + num2')
console.log(code({ num1: 10, num2: 7 }))
// 输出17
const globalNum = 12
const otherCode = compileCode('return globalNum')
console.log(otherCode({ num1: 2, num2; 3 }))
// 全局作用域是被保护的
// 输出 undefined
```

在文章的最后，我们将会用少于20行代码实现`compileCode()`函数。

## new Function()
> 这个构造函数用于创建函数对象。在 JavaScript 中，每个函数其实都是一个函数对象。

这个 `Function` 就是 `eval()` 的替代方案。`new Function(...args, 'funcBody')` 以代码形式运算传递进来的 `funcBody` 字符串，并且返回一个函数用于执行解析完的代码。它和 `eval()` 不同的地方在于两个主要地方：

- 传递进来的代码它进行一次运算。调用返回的函数也不会再次运算，只是执行这段代码。
- 它不能访问本地闭包内的变量，但是，它可以访问全局作用域。

```JavaScript
function compileCode(src) {
  return new Function(src);
}
```

对于 `eval()`, `new Function()` 在我们的案例中是一个更好的替代方案。在性能和安全上它表现得更加优秀，但是为了让它真的可用，全局作用域还是需要被保护起来的。

## With 关键字
> with 语句用于扩展一个语句的作用域链

`with` 是个鲜为人知的 JavaScript 关键字。它允许在半沙箱环境下的执行。在 `with` 的块作用域下的代码会首先尝试从传递进来的对象上检索变量，但是如果它从里面找不到，它会去闭包作用域和全局作用域中寻找，闭包作用域被 `new Function()` 保护起来了，所以我们只需要担心全局作用域了。

```JavaScript
function compileCode(src) {
  src = 'with (sandbox) {' + src + '}'
  return new Function('sandbox', src)
}
```

`with` 在内部使用了 `in` 操作符。在块作用域里面的每个变量都会执行 `variable in sandbox` 的判断。如果判断为真，它从 sandbox 中检索出变量。否则，它会从全局作用域中寻找。通过糊弄 `with` 让它每次执行 `variable in sandbox` 都为真值，从而阻止它访问全局作用域。

![](http://blog-assets.risingstack.com/2016/Aug/Sandboxed_code_evaluation_simple_with_statement-1470403007416.svg)

## ES6 Proxy
> Proxy 对象用于定义基本操作的自定义行为 (例如, 属性查找，赋值，枚举，函数调用,等)。

ES6 的 Proxy 会包裹一个对象并且定义代理函数，代理函数可能会拦截一个对象的基本操作。在一个基本操作触发时会调用代理函数。通过用 `Proxy` 包裹 `sandbox` 对象和定义 `has` 代理，我们可以重写默认的 `in` 操作符的行为。

```JavaScript
function compileCode(src) {
  src = 'with (sandbox) {' + src + '}'
  const code = new Function('sandbox', src)

  return function(sandbox) {
    const sandboxProxy = new Proxy(sandbox, { has })
    return code(sandboxProxy)
  }
}

function has(target, key) {
  return has
}
```

上述的代码糊弄了 `with` 块级作用域。`variable in sandbox` 将会总是执行结果为真，因为 `has` 被代理成总是返回真。在 `with` 块级作用域内的代码就绝不会访问到全局作用域了。

![](http://blog-assets.risingstack.com/2016/Aug/Sandboxed_code_evaluation_with_statement_and_proxies-1470403030877.svg)

## Symbol.unscopables
> Symbol 是一种唯一且不可变的数据类型，可以用作对象的属性。

`Symbol.unscopables` 是一种符号。一个用 JavaScript `Symbol` 构建的，代表语言的内部行为。比如符号可以用于添加或者重写遍历或者原始的换行行为。

> Symbol.unscopables 指用于指定对象值，其对象自身和继承的从关联对象的 with 环境绑定中排除的属性名称。

`Symbol.unscopable` 用于定义一个对象的不可访问属性。在 `with` 块级作用域中是不能访问到 sandbox 内的不可能访问属性的，但是闭包作用域和全局作用域是可以直接访问的。`Symbol.unscopable` 是一种十分少用的特性。可以读这篇[文章](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/unscopables)了解原因。

![](http://blog-assets.risingstack.com/2016/Aug/Sandboxed_code_evaluation_security_issue-1470403047129.svg)

我们可以修复这个问题，通过定义一个 `get` 代理函数到 sandbox 的 `Proxy` 上，它拦截 `Symbol.unscopables` 的检索，并且总是返回 `undefined`。这可以糊弄 `with` 块级作用域认为我们的 sandbox 对象上没有这个属性。

```JavaScript
function compileCode(src) {
  src = 'with (sandbox) {' + src + '}'
  const code = new Function('sandbox', src)

  return function(sandbox) {
    const sandProxy = new Proxy(sandbox, { has, get })
    return code(sandbox)
  }
}

function has(target, key) {
  return true
}

function get(target, key) {
  if (key === Symbol.unscopables) return undefined
  return target[key]
}
```

![](http://blog-assets.risingstack.com/2016/Aug/with_statements_and_proxies_has_and_get_traps-1470403073125.svg)

## 用于缓存的 WeakMap
现在的代码安全性是 👌 的，但是性能还有待提升，因为它在每次调用返回新的函数时都要创新一个代理对象。这个可以用缓存来解决，让每次用同一个沙箱的函数在调用的时用同一个 `Proxy`。

一个代理属于一个沙箱对象，所以我们可以简单的添加代理到沙箱对象上作为属性值。尽管如此，这样的实现会暴露我们的内部细节，并且在沙箱对象被 `Object.freeze()` 处理成不可变对象的时候就失效了。在这样的情况下，用 `WeakMap` 是一种更好的选择。

> WeakMap 对象是一组键/值对的集合，且其中的键是弱引用的。键必须是对象，值可以是任意值。

WeakMap 可以不需要扩展属性就把一份数据放到一个对象上。我们可以用 `WeakMap` 直接把沙箱对象放到 `Proxies` 缓存里面去。

```JavaScript
const sandboxProxies = new WeakMap()

function compileCode (src) {  
  src = 'with (sandbox) {' + src + '}'
  const code = new Function('sandbox', src)

  return function (sandbox) {
    if (!sandboxProxies.has(sandbox)) {
      const sandboxProxy = new Proxy(sandbox, {has, get})
      sandboxProxies.set(sandbox, sandboxProxy)
    }
    return code(sandboxProxies.get(sandbox))
  }
}

function has (target, key) {  
  return true
}

function get (target, key) {  
  if (key === Symbol.unscopables) return undefined
  return target[key]
}
```

这样每个沙箱对象就只会创建一次 `Proxy`。

## 最后的提醒
上述的 `compileCode()` 案例是一个可用沙箱求值代码，仅仅只用了 19 行代码。如果你想看完整的 nx-compile 库，可以在[这里](https://github.com/RisingStack/nx-compile)找到。

抛开代码执行的解释，这一章节的目标是展示 ES6 的新属性是如何修复一些已存在的问题，而不是重写他们。我试着通过这个案例证明 `Proxy` 和 `Symbol` 的能力有多大。

> [原文](https://blog.risingstack.com/writing-a-javascript-framework-sandboxed-code-evaluation/)
