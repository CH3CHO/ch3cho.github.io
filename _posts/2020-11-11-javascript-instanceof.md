---
layout: article
title: 从一个问题来理解 JS 中 instanceof 的实现方式
key: article-javascript-instanceof
---
### 问题出现

这周，在对接一个第三方组件的时候，我遇到了一个问题。问题本身比较复杂，这里就不展开了。排查到最后，我的注意力集中在了下面这段代码上：

```typescript
const xhr = new XMLHttpRequest();
// 此处对xhr的初始化逻辑省略
if (responseType !== 'text' && xhr instanceof XMLHttpRequest) {
    xhr.responseType = responseType;
}
```

通过调试，我发现 `xhr instanceof XMLHttpRequest` 的值居然是 `false`！作为一名 Java 程序员，看到这个的时候我表示震惊。

随后，我谷歌了 JavaScript 中 `instanceof` 运算符的实现方式，再翻了一下我们项目的代码，终于理解了导致这个问题的原因。

### 原型链继承

JavaScript 中的每一个对象实例中都有一个私有属性，名为 `__proto__`，指向它的构造函数的原型对象（prototype）。这个原型对象也有一个自己的原型对象（`__proto__`）。这样一层一层上去直到最顶层，它的原型对象为 `null`。根据定义，`null `没有原型，并作为这个原型链中的最后一个环节。

### instanceof 的工作方式

按照MDN上的说明，`instanceof` 运算符的工作方式是检测构造函数的 `prototype` 属性是否出现在某个实例对象的原型链上。也就是说，它会逐层遍历实例的原型链，判断各层的`__proto__`属性的值是否与指定类型的`prototype`属性的值相同。如果相同的话，则返回`true`，反之则返回`false`。

### 问题原因揭秘

综合以上内容，JavaScript 中对象的“继承”关系是靠这个原型链来记录的。如果原型链被修改了，那么之前的对象继承关系也就被破坏了。

我们的项目中引用了另外一个第三方库，其中有这么一段代码：

```javascript
var _XMLHttpRequest = window.XMLHttpRequest;
window.XMLHttpRequest = function(flags) {
    var req;
    req = new _XMLHttpRequest(flags);
    monitorXHR(req);
    return req;
};
```

这里的代码对 `XMLHttpRequest` 进行了封装，虽然返回的还是原始的 `XMLHttpRequest` 对象，但 `window.XMLHttpRequest` 已经是封装之后的函数了，二者的原型链不一致，所以 `instanceof` 会返回 `false`。

由于这个第三方库的功能目前已经不再使用，我们将其从项目中移除。移除后，对接的第三方组件就可以正常工作了。
