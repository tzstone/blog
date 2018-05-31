# 单例模式

定义: 保证一个类仅有一个实例, 并提供一个访问它的全局访问点.

实现: 用一个变量来标志当前是否已经为某个类创建过对象, 如果是, 则在下一次获取该类的实例时, 直接返回之前创建的对象.

```javascript
var Singleton = (function() {
  var instance;

  return function() {
    if (instance) {
      return instance;
    }
    // ...
    return instance = this;
  }
})()
```

另一种实现:

```javascript
function Singleton() {
  if (typeof Singleton.instance === 'object') {
    return Singleton.instance;
  }
  // ...
  return Singleton.instance = this;
}
```

使用代理类: 保持`Singleton`类的职责单一

```javascript
function Singleton() {
  // ...
}

var ProxySingleton = (function() {
  var instance;

  return function() {
    if (!instance) {
      instance = new Singleton();
    }

    return instance
  }
})()
```

惰性单例: 在需要的时候才创建对象实例

通用的惰性单例: 分离创建实例对象的职责和管理单例的职责

```javascript
var getSingle = function(fn) {
  var result;
  return function() {
    return result || (result = fn.apply(this, arguments))
  }
}
```

摘自[javascript设计模式与开发实践](https://book.douban.com/subject/26382780/)
