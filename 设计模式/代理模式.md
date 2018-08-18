# 代理模式

定义: 代理模式是为一个对象提供一个代用品或占位符, 以便控制对它的访问.

## 保护代理

控制不同权限的对象对目标对象的访问

## 虚拟代理

把一些开销很大的对象, 延迟到真正需要它的时候才去创建.

- 图片预加载

```javascript
var myImage = (function() {
  var imgNode = document.createElement("img");
  document.body.appendChild(imgNode);

  return {
    setSrc: function(src) {
      imgNode.src = src;
    }
  };
})();

var proxyImage = (function() {
  var img = new Image();
  img.onload = function() {
    myImage.setSrc(this.src);
  };

  return {
    setSrc: function(src) {
      myImage.setSrc("../loading.gif");
      img.src = src;
    }
  };
})();

proxyImage.setSrc("http://....a.jpg");
```

- 合并 http 请求

```javascript
var synchronousFile = function(id) {
  console.log("开始同步文件, id:", id);
};

var proxySynchronousFile = (function() {
  var cache = [],
    timer;

  return function(id) {
    cache.push(id);
    if (timer) return;

    timer = setTimeout(function() {
      synchronousFile(cache.join(","));
      clearTimeout(timer);
      timer = null;
      cache.length = 0; // 清空ID集合
    }, 2000);
  };
})();

var checkbox = document.getElementsByTagName("input");
for (var i = 0, c; (c = checkbox[i++]); ) {
  c.onclick = function() {
    if (this.checked === true) {
      proxySynchronousFile(this.id);
    }
  };
}
```

- 惰性加载

```javascript
// 在真正加载js前, 提供一个占位的代理对象给用户提前使用
var miniConsole = (function() {
  var cache = [];
  var handler = function(ev) {
    // 按下F2, 开始加载js
    if (ev.keyCode === 13) {
      var script = document.createElement("script");
      script.onload = function() {
        for (var i = 0, fn; (fn = cache[i++]); ) {
          fn();
        }
      };
      script.src = "miniConsole.js";
      document.getElementByTagName("head")[0].appendChild(script);
      document.body.removeEventListener("keydown", handler); // 只加载一次js
    }
  };

  document.body.addEventListener("keydown", handler, false);

  return {
    log: function() {
      var args = arguments;
      cache.push(function() {
        return miniConsole.log.apply(miniConsole, args);
      });
    }
  };
})();

miniConsole.log(11); // 开始打印log

// miniConsole.js代码
miniConsole = {
  log: function() {
    // ...
    console.log(Array.prototype.join.call(arguments));
  }
};
```

## 缓存代理

缓存代理可以为一些开销大的运算结果提供暂时的存储, 在下次运算时, 如果传递进来的参数跟之前一致, 则可以直接返回前面存储的运算结果.

```javascript
var mult = function() {
  var a = 1;
  for (var i = 0, l = arguments.length; i < l; i++) {
    a = a * arguments[i];
  }
  return a;
};

var proxyMult = (function() {
  var cache = {};
  return function() {
    var args = Array.prototype.join.call(arguments, ",");
    if (args in cache) {
      return cache[args];
    }
    return (cache[args] = mult.apply(this, arguments));
  };
})();
```

## 代理的意义

添加代理对象使得接口行为符合单一职责原则

- 单一职责原则

  就一个类(通常也包括对象和函数等)而言, 应该仅有一个引起它变化的原因. 职责被定义为"引起变化的原因". 如果一个对象承担的职责过多, 等于把这些职责耦合到了一起, 这种耦合会导致脆弱和低内聚的设计.

## 代理和本体接口应保持一致性

- 用户可以放心地请求代理, 他只关心是否能得到想要的结果
- 在任何使用本体的地方都可以替换成使用代理

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
