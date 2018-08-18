# 发布-订阅模式

发布-订阅模式又叫观察者模式, 它定义对象间的一种一对多的依赖关系, 当一个对象的状态发生改变时, 所有依赖于它的对象都将得到通知.

## 优点

- 时间上解耦(异步编程)
- 对象之间解耦(取消对象之间硬编码的通知机制)

## 实现

- 首先指定好谁充当发布者
- 给发布者添加一个缓存列表, 用于存放回调函数以便通知订阅者
- 发布消息时, 发布者会遍历这个缓存列表, 依次触发里面存放的订阅者回调函数

```javascript
var event = {
  // 缓存列表
  clientList: {},
  // 添加订阅者
  listen: function(key, fn) {
    if (!this.clientList[key]) {
      this.clientList[key] = [];
    }
    this.clientList[key].push(fn); // 订阅的消息添加进缓存列表
  },
  // 发布消息
  trigger: function() {
    var key = Array.prototype.shift.call(arguments),
      fns = this.clientList[key];

    if (!fns || fns.length === 0) return false;

    for (var i = 0, fn; (fn = fns[i++]); ) {
      fn.apply(this, arguments);
    }
  },
  // 取消订阅事件
  remove: function(key, fn) {
    var fns = this.clientList[key];

    if (!fns) return false;

    // 没有传入具体的回调函数, 则取消key对应的所有订阅事件
    if (!fn) {
      fns && (fns.length = 0);
    } else {
      // 反向遍历, 避免当存在多个匹配的fn时splice带来的数组乱序
      for (var l = fns.length - 1; l >= 0; l--) {
        var _fn = fns[l];
        if (_fn === fn) {
          fns.splice(l, 1);
        }
      }
    }
  }
};
```

`注意`: 如果模块之间大量使用全局发布-订阅模式来通信, 那么模块与模块之间的联系就被隐藏了, 我们很难知道消息来自哪个模块, 或者消息会流向哪些模块, 这会让维护变得困难.

## 先发布再订阅

在某些应用场景中, 我们可能会在发布消息后才去订阅, 为了让后来的订阅者也能接收到消息. 我们要建立一个存放离线事件的堆栈, 当事件发布时, 如果此时还没有订阅者来订阅这个事件, 则暂时把发布事件的动作包裹在一个函数里, 这些包装函数将被存入堆栈中, 等到终于有对象来订阅此事件时, 遍历堆栈并依次执行这些包装函数, 也就是重新发布里面的事件.

## 全局事件的命名冲突

全局的发布-订阅对象中只有 clientList 对象来存放消息名和回调函数, 容易出现事件名冲突的情况, 可以通过给 event 对象提供创建命名空间的功能来解决这一问题.

```javascript
// 最终代码
var Event = (function() {
  var global = this,
    Event,
    _default = "default";

  Event = (function() {
    var _listen,
      _trigger,
      _remove,
      _slice = Array.prototype.slice,
      _shift = Array.prototype.shift,
      _unshift = Array.prototype.unshift,
      namespaceCache = {},
      _create,
      find,
      each = function(ary, fn) {
        var ret;
        for (var i = 0, l = ary.length; i < l; i++) {
          var n = ary[i];
          ret = fn.call(n, i, n);
        }
        return ret;
      };

    _listen = function(key, fn, cache) {
      if (!cache[key]) {
        cache[key] = [];
      }
      cache[key].push(fn);
    };

    _remove = function(key, cache, fn) {
      if (cache[key]) {
        if (fn) {
          for (var i = cache[key].length; i >= 0; i--) {
            if (cache[key][i] === fn) {
              cache[key].splice(i, 1);
            }
          }
        } else {
          cache[key] = [];
        }
      }
    };

    _trigger = function() {
      var cache = _shift.call(arguments),
        key = _shift.call(arguments),
        args = arguments,
        _self = this,
        ret,
        stack = cache[key];

      if (!stack || !stack.length) return;

      return each(stack, function() {
        return this.apply(_self, args);
      });
    };

    // 创建一个带命名空间的全局发布-订阅对象
    _create = function(namespace) {
      var namespace = namespace || _default;
      var cache = {},
        offlineStack = [], // 离线事件
        ret = {
          listen: function(key, fn, last) {
            _listen(key, fn, cache);

            // 已经订阅过事件
            if (offlineStack === null) return;

            if (last === "last") {
              offlineStack.length && offlineStack.pop()();
            } else {
              each(offlineStack, function() {
                this();
              });
            }

            // 第一次订阅事件后把offlineStack设置为null, 不再使用offlineStack
            offlineStack = null;
          },
          one: function(key, fn, last) {
            _remove(key, cache);
            this.listen(key, fn, last);
          },
          remove: function(key, fn) {
            _remove(key, cache, fn);
          },
          trigger: function() {
            var fn,
              args,
              _self = this;

            _unshift.call(arguments, cache);
            args = arguments;
            fn = function() {
              return _trigger.apply(_self, args);
            };

            // 如果offlineStack不为null, 说明此时还没订阅事件, 则把发布行为保存在offlineStack
            if (offlineStack) {
              return offlineStack.push(fn);
            }

            return fn();
          }
        };

      return namespace
        ? namespaceCache[namespace]
          ? namespaceCache[namespace]
          : (namespaceCache[namespace] = ret)
        : ret;
    };

    return {
      create: _create,
      one: function(key, fn, last) {
        var event = this.create();
        event.one(key, fn, last);
      },
      remove: function(key, fn) {
        var event = this.create();
        event.remove(key, fn);
      },
      listen: function(key, fn, last) {
        // 创建一个发布-订阅对象, 因为没指定命名空间, 默认都是default命名空间
        var event = this.create();
        event.listen(key, fn, last);
      },
      trigger: function() {
        // 同上
        var event = this.create();
        event.trigger.apply(this, arguments);
      }
    };
  })();

  return Event;
})();
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
