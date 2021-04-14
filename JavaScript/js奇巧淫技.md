# js 奇巧淫技合集

- 判断一个数是否为正整数

```js
var number = '10.6';

// 方法一
((number - 0) | 0) === number - 0;

// 方法二, es6支持
Number.isInteger(number - 0);
```

- 浮点数取整

```js
const x = 123.545;
x >> 0; // 123
~~x; // 123
x | 0; // 123
Math.floor(x); // 123
```

注意：前三种方法只适用于 32 个位整数, 对于负数的处理上和 Math.floor 是不同的.

```js
const x = -123.545;
x >> 0; // -123
~~x; // -123
x | 0; // -123
Math.floor(x); // -124
```

- 16 进制颜色代码生成

```js
(function() {
  return (
    '#' + ('00000' + ((Math.random() * 0x1000000) << 0).toString(16)).slice(-6)
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

query('?key1=value1&key2=value2');
```

- 获取 URL 参数

```js
function getQueryString(key) {
  var reg = new RegExp('(^|&)' + key + '=([^&]*)(&|$)');
  var r = window.location.search.substr(1).match(reg);

  if (r != null) {
    return unescape(r[2]);
  }

  return null;
}
```

- n 维数组展开成一维数组

```js
var foo = [1, [2, 3], ['4', 5, ['6', 7, [8]]], [9], 10];

// 方法一
// 限制：数组项不能出现`,`, 同时数组项全部变成了字符数字
foo.toString().split(','); // ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10"]

// 方法二
// 转换后数组项全部变成数字了
eval('[' + foo + ']'); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// 方法三, 使用ES6展开操作符
// 写法太过麻烦, 太过死板
[1, ...[2, 3], ...['4', 5, ...['6', 7, ...[8]]], ...[9], 10]; // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]

// 方法四
JSON.parse(`[${JSON.stringify(foo).replace(/\[|]/g, '')}]`); // [1, 2, 3, "4", 5, "6", 7, 8, 9, 10]

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
  return !/^.?$|^(..+?)\1+$/.test('1'.repeat(n));
}
```

- 统计字符串中相同字符出现的次数

```js
var arr = 'abcdaabc';

var info = arr.split('').reduce((p, k) => (p[k]++ || (p[k] = 1), p), {});

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

// 方法一
a ^= b;
b ^= a;
a ^= b;

// 方法二
[a, b] = [b, a];

// 方法三
b = [a, (a = b)][0];

// 方法四
a = a + b;
b = a - b;
a = a - b;
```

- 数字字符转数字

```js
var a = '1';
+a; // 1
a * 1; // 1
```

- 最短的代码实现数组去重

```js
[...new Set([1, '1', 2, 1, 1, 3])]; // [1, "1", 2, 3]
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
var str = 'hello world';
if (str.indexOf('lo') > -1) {
  // ...
}

if (~str.indexOf('lo')) {
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
    throw 'Peron must called with new';
  }
  this.name = name;
  this.age = age;
}
```

- 使用 Boolean 过滤数组中的所有假值

```js
const compact = arr => arr.filter(Boolean);
compact([0, 1, false, 2, '', 3, 'a', 'e' * 23, NaN, 's', 34]); // [ 1, 2, 3, 'a', 's', 34 ]
```

- 短路运算符

  - `&&` 为取假运算, 从左到右依次判断, 如果遇到一个假值, 就返回假值, 以后不再执行, 否则返回最后一个真值
  - `||` 为取真运算, 从左到右依次判断, 如果遇到一个真值, 就返回真值, 以后不再执行, 否则返回最后一个假值

```js
var a = undefined && true; // undefined
var b = 3 || undefined; // 3
```

- 判断奇偶数

```js
const num = 3;
!!(num & 1); // true
!!(num % 2); // true
```

- 字符串比较时间先后

字符串比较大小是按照字符串从左到右每个字符的 charCode 来的, 但所以特别要注意时间形式, 注意补 0

```js
var a = '2014-08-08';
var b = '2014-09-09';
console.log(a > b, a < b); // false true
console.log('21:00' < '09:10'); // false
console.log('21:00' < '9:10'); // true   时间形式注意补0
```

- 精确到指定位数的小数(四舍五入)

将数字四舍五入到指定的小数位数. 使用 Math.round() 和模板字面量将数字四舍五入为指定的小数位数. 省略第二个参数 decimals, 数字将被四舍五入到一个整数.

```js
const round = (n, decimals = 0) =>
  Number(`${Math.round(`${n}e${decimals}`)}e-${decimals}`);
round(1.345, 2); // 1.35
round(1.345, 1); // 1.3
```

- 数字补 0

有时候比如显示时间的时候有时候会需要把一位数字显示成两位, 这时候就需要补 0 操作, 可以使用 slice 和 string 的 padStart 方法

```js
const addZero1 = (num, len = 2) => `0${num}`.slice(-len);
const addZero2 = (num, len = 2) => `${num}`.padStart(len, '0');
addZero1(3); // 03
addZero2(32, 4); // 0032
```

- reduce 方法同时实现 map 和 filter

假设现在有一个数列, 你希望更新它的每一项（map 的功能）然后筛选出一部分(filter 的功能). 如果是先使用 map 然后 filter 的话, 你需要遍历这个数组两次. 在下面的代码中, 我们将数列中的值翻倍, 然后挑选出那些大于 50 的数.

```js
const numbers = [10, 20, 30, 40];
const doubledOver50 = numbers.reduce((finalList, num) => {
  num = num * 2;
  if (num > 50) {
    finalList.push(num);
  }
  return finalList;
}, []);
doubledOver50; // [60, 80]
```

- 将数组平铺到指定深度

使用递归, 为每个深度级别 depth 递减 1 . 使用 Array.reduce() 和 Array.concat() 来合并元素或数组. 基本情况下, depth 等于 1 停止递归. 省略第二个参数, depth 只能平铺到 1 (单层平铺) 的深度.

```js
const flatten = (arr, depth = 1) =>
  depth != 1
    ? arr.reduce(
        (a, v) => a.concat(Array.isArray(v) ? flatten(v, depth - 1) : v),
        []
      )
    : arr.reduce((a, v) => a.concat(v), []);
flatten([1, [2], 3, 4]); // [1, 2, 3, 4]
flatten([1, [2, [3, [4, 5], 6], 7], 8], 2); // [1, 2, 3, [4, 5], 6, 7, 8]
```

- 使用解构删除不必要属性

有时候你不希望保留某些对象属性, 也许是因为它们包含敏感信息或仅仅是太大了(just too big). 你可能会枚举整个对象然后删除它们, 但实际上只需要简单的将这些无用属性赋值给变量, 然后把想要保留的有用部分作为剩余参数就可以了.

下面的代码里, 我们希望删除\_internal 和 tooBig 参数. 我们可以把它们赋值给 internal 和 tooBig 变量, 然后在 cleanObject 中存储剩下的属性以备后用.

```js
let { _internal, tooBig, ...cleanObject } = {
  el1: '1',
  _internal: 'secret',
  tooBig: {},
  el2: '2',
  el3: '3',
};
console.log(cleanObject);
```

- 在函数参数中解构嵌套对象

在下面的代码中, engine 是对象 car 中嵌套的一个对象. 如果我们对 engine 的 vin 属性感兴趣, 使用解构赋值可以很轻松地得到它.

```js
var car = {
  model: 'bmw 2018',
  engine: {
    v6: true,
    turbo: true,
    vin: 12345,
  },
};
const modelAndVIN = ({ model, engine: { vin } }) => {
  console.log(`model: ${model} vin: ${vin}`);
};
modelAndVIN(car); // => model: bmw 2018  vin: 12345
```

- 强制参数

默认情况下, 如果不向函数参数传值, 那么 JS 会将函数参数设置为 undefined. 其它一些语言则会发出警告或错误. 要执行参数分配, 可以使用 if 语句抛出未定义的错误, 或者可以利用强制参数.

```js
mandatory = () => {
  throw new Error('Missing parameter!');
};
foo = (bar = mandatory()) => {
  // 这里如果不传入参数, 就会执行manadatory函数报出错误
  return bar;
};
```

## 参考资料

- [JavaScript 有用的代码片段和 trick](https://mp.weixin.qq.com/s/Z-Vcfl1D5oKMLliGTVhE1g)
- [JS 中可以提升幸福度的小技巧](https://mp.weixin.qq.com/s/J0b52s9Qt8ZEfTF6Wv_GZw)
