# js 奇巧淫技合集

- 判断一个数是否为正整数

```js
var number = "10.6";

// 方法一
((number - 0) | 0) === number - 0;

// 方法二, es6支持
Number.isInteger(number - 0);
```

- 浮点数取整

```js
const x = 123.4545;
x >> 0; // 123
~~x; // 123
x | 0; // 123
Math.floor(x); // 123
```

注意：前三种方法只适用于 32 个位整数，对于负数的处理上和 Math.floor 是不同的。

```js
Math.floor(-12.53); // -13
-12.53 | 0; // -12
```

- 16 进制颜色代码生成

```js
(function() {
  return (
    "#" + ("00000" + ((Math.random() * 0x1000000) << 0).toString(16)).slice(-6)
  );
})();
```

- url 查询参数转 json 格式

```js
var query = function() {
  var o = {},
    search = arguments[0] || window.location.search;
  search.replace(/([^?=&]+)=([^&]*)/g, function(m, key, value) {
    o[key] = decodeURIComponent(value);
  });
  return o;
};

query("?key1=value1&key2=value2");
```

- 获取 URL 参数

```js
function getQueryString(key) {
  var reg = new RegExp("(^|&)" + key + "=([^&]*)(&|$)");
  var r = window.location.search.substr(1).match(reg);

  if (r != null) {
    return unescape(r[2]);
  }

  return null;
}
```

- n 维数组展开成一维数组

```js
var foo = [1, [2, 3], ["4", 5, ["6", 7, [8]]], [9], 10];

// 方法一
// 限制：数组项不能出现`,`，同时数组项全部变成了字符数字
foo.toString().split(","); // ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10"]

// 方法二
// 转换后数组项全部变成数字了
eval("[" + foo + "]"); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// 方法三，使用ES6展开操作符
// 写法太过麻烦，太过死板
[1, ...[2, 3], ...["4", 5, ...["6", 7, ...[8]]], ...[9], 10]; // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]

// 方法四
JSON.parse(`[${JSON.stringify(foo).replace(/\[|]/g, "")}]`); // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]
// 方法五
const flatten = ary =>
  ary.reduce((a, b) => a.concat(Array.isArray(b) ? flatten(b) : b), []);
flatten(foo); // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]

// 方法六
function flatten(a) {
  return Array.isArray(a) ? [].concat(...a.map(flatten)) : a;
}
flatten(foo); // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]
```

- 测试质数

```js
function isPrime(n) {
  return !/^.?$|^(..+?)\1+$/.test("1".repeat(n));
}
```

- 统计字符串中相同字符出现的次数

```js
var arr = "abcdaabc";

var info = arr.split("").reduce((p, k) => (p[k]++ || (p[k] = 1), p), {});

console.log(info); //{ a: 3, b: 2, c: 2, d: 1 }
```

- 使用 void0 来解决 undefined 被污染问题

```js
undefined = 1;
!!undefined; // true
!!void 0; // false
```

- 匿名函数自执行写法

```html
(function() {}());
(function() {})();
[function() {}()];

~function() {}();
!function() {}();
+function() {}();
-function() {}();

delete function() {}();
typeof function() {}();
void function() {}();
new function() {}();
new function() {};

var f = function() {}();

1, function() {}();
1 ^ function() {}();
1 > function() {}();
```

- 两个整数交换数值

```js
var a = 20,
  b = 30;

a ^= b;
b ^= a;
a ^= b;
a; // 30
b; // 20
```

- 数字字符转数字

```js
var a = "1";
+a; // 1
```

- 最短的代码实现数组去重

```js
[...new Set([1, "1", 2, 1, 1, 3])]; // [1, "1", 2, 3]
```

- 用最短的代码实现一个长度为 m(6)且值都 n(8)的数组

```js
Array(6).fill(8); // [8, 8, 8, 8, 8, 8]
```

- 将 argruments 对象转换成数组

```js
var argArray = Array.prototype.slice.call(arguments);

// ES6：
var argArray = Array.from(arguments);

// or
var argArray = [...arguments];
```

- 使用 ~x.indexOf('y')来简化 x.indexOf('y')>-1

```js
var str = "hello world";
if (str.indexOf("lo") > -1) {
  // ...
}

if (~str.indexOf("lo")) {
  // ...
}
```

- 判断对象的实例

```js
// 方法一: ES3
function Person(name, age) {
  if (!(this instanceof Person)) {
    return new Person(name, age);
  }

  this.name = name;
  this.age = age;
}

// 方法二: ES5
function Person(name, age) {
  var self = this instanceof Person ? this : Object.create(Person.prototype);
  self.name = name;
  self.age = age;
  return self;
}

// 方法三：ES6
function Person(name, age) {
  if (!new.target) {
    throw "Peron must called with new";
  }
  this.name = name;
  this.age = age;
}
```

## 参考资料

[JavaScript 有用的代码片段和 trick](https://mp.weixin.qq.com/s/Z-Vcfl1D5oKMLliGTVhE1g)
