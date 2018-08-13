# 职责链模式

使多个对象都有机会处理请求, 从而避免请求的发送者和接收者之间的耦合关系, 将这些对象连成一条链, 并沿着这条链传递该请求, 直到有一个对象处理它为止.

before:

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/%E8%81%8C%E8%B4%A3%E9%93%BE-before.jpeg" align=center width=500/>

after:

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/%E8%81%8C%E8%B4%A3%E9%93%BE-after.jpeg" align=center width=500/>

## 优点

- 解耦了请求发送者和 N 个请求接收者之间的复杂关系
- 链中的节点对象可以灵活地拆分重组
- 可以手动指定起始节点

## 缺点

- 不能保证请求一定会被链中的节点处理(可以在链尾添加一个保底的接受者节点)
- 在某一次请求传递中, 大部分节点并没有实质性的作用, 只是让请求传递下去. 需要规避过长的职责链带来的性能损耗

```javascript
var Chain = function(fn) {
  this.fn = fn;
  this.successor = null;
};
// 指定职责链中的下一个节点
Chain.prototype.setNextSuccessor = function(successor) {
  return (this.successor = successor);
};
// 传递请求给某个节点, 同步
Chain.prototype.passRequest = function() {
  var ret = this.fn.apply(this, arguments);

  if (ret === "nextSuccessor") {
    return (
      this.successor &&
      this.successor.passRequest.apply(this.successor, arguments)
    );
  }

  return ret;
};
// 用于异步中手动传递请求
Chain.prototype.next = function() {
  return (
    this.successor &&
    this.successor.passRequest.apply(this.successor, arguments)
  );
};

// 包装职责链的节点
var fn1 = new Chain(function() {
  console.log(1);
  return "nextSuccessor";
});

var fn2 = new Chain(function() {
  console.log(2);
  var self = this;
  // 异步中传递请求
  setTimeout(function() {
    self.next();
  }, 1000);
});

var fn3 = new Chain(function() {
  console.log(3);
});

// 设置节点顺序
fn1.setNextSuccessor(fn2).setNextSuccessor(fn3);

// 把请求传递给第一个节点
fn1.passRequest();
```

## 用 AOP 实现职责链

把函数叠在一起, 同时也叠加了函数的作用域, 如果链条太长, 对性能有较大影响

```javascript
var order500 = function(orderType, pay, stock) {
  if (orderType === 1 && pay === true) {
    console.log("500元定金预购, 得到100优惠券");
  } else {
    return "nextSuccessor"; // 把请求往后传递
  }
};
var order200 = function(orderType, pay, stock) {
  if (orderType === 2 && pay === true) {
    console.log("200元定金预购, 得到50优惠券");
  } else {
    return "nextSuccessor"; // 把请求往后传递
  }
};
var orderNormal = function(orderType, pay, stock) {
  if (stock > 0) {
    console.log("普通购买, 无优惠券");
  } else {
    console.log("手机库存不足");
  }
};

Function.prototype.after = function(fn) {
  var self = this;
  return function() {
    var ret = self.apply(this, arguments);
    if (ret === "nextSuccessor") {
      return fn.apply(this, arguments);
    }

    return ret;
  };
};

var order = order500.after(order200).after(orderNormal);

order(1, true, 500);
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
