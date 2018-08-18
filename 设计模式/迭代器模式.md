# 迭代器模式

迭代器模式是指提供一种方法顺序访问一个聚合对象中的各个元素, 而又不需要暴露该对象的内部表示. 迭代器模式可以把迭代的过程从业务逻辑中分离出来, 在使用迭代器模式之后, 即使不关心对象的内部构造, 也可以按顺序访问其中的每个元素.

## 内部迭代器

内部迭代器在内部已经定义好了迭代规则, 它完全接手整个迭代过程, 外部只需要一次初始调用.

```javascript
var each = function(ary, callback) {
  for (var i = 0, l = ary.length; i < l; i++) {
    callback.call(ary[i], i, ary[i]);
  }
};

each([1, 2, 3], function(i, n) {
  alert(i, n);
});
```

## 外部迭代器

外部迭代器必须显式地请求迭代下一个元素, 这增加了调用的复杂度, 但相对地也增强了迭代器的灵活性, 我们可以手工控制迭代的过程或者顺序.

```javascript
var Iterator = function(obj) {
  var current = 0;
  var next = function() {
    current += 1;
  };
  var isDone = function() {
    return current >= obj.length;
  };
  var getCurrItem = function() {
    return obj[current];
  };
  return {
    next: next,
    isDone: isDone,
    getCurrItem: getCurrItem,
    length: obj.length
  };
};

var iterator = Iterator([1, 2, 3]);
```

## 迭代类数组对象和字面量对象

- 迭代器可以迭代一些类数组对象, 如 arguments, {"0": "a", "1": "b"}. 只要被迭代的聚合对象拥有 length 属性而且可以用下标访问, 就可以被迭代.
- 对于普通字面量对象, 可以通过`for in`语句进行迭代.

## 中止迭代器

迭代器可以像普通 for 循环中的 break 一样, 提供一种跳出循环的方法.

```javascript
var each = function(ary, callback) {
  for (var i = 0, l = ary.length; i < l; i++) {
    if (callback(i, ary[i]) === false) {
      break;
    }
  }
};

each([1, 2, 3, 4, 5], function(i, n) {
  if (n > 3) {
    return false;
  }
  console.log(n);
});
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
