# js 奇巧淫技合集

- 判断一个数是否为正整数

```js
var number = "10.6";

// 方法一
((number - 0) | 0) === number - 0;

// 方法二, es6支持
Number.isInteger(number - 0);
```
