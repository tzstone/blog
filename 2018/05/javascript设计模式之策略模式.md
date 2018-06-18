# 策略模式

定义: 定义一系列的算法, 把它们一个个封装起来, 并且使它们可以相互替换.

实现: 一个基于策略模式的程序至少由两部分组成. 第一个部分是一组策略类, 策略类封装了具体的算法, 并负责具体的计算过程. 第二个部分是环境类 Context, Context 接收客户的请求, 随后把请求委托给某一个策略类.

```javascript
var strategies = {
  S: function(salary) {
    return salary * 4
  },
  A: function(salary) {
    return salary * 3
  },
  B: function(salary) {
    return salary * 2
  }
}

var calculateBonus = function(level, salary) {
  return strategies[level](salary)
}

console.log(calculateBonus('S', 20000))
console.log(calculateBonus('A', 10000))
```

## 优缺点

- 策略模式利用组合、委托和多态等技术和思想，可以有效地避免多重条件选择语句。
- 策略模式提供了对开放-封闭原则的完美支持，将算法封装在独立的 strategy 中，使得它们易于切换，易于理解，易于扩展。
- 策略模式中的算法也可以复用在系统的其他地方，从而避免许多重复的复制粘贴工作。
- 在策略模式中利用组合和委托来让 context 拥有执行算法的能力，这也是继承的一种更轻便的替代方案。

- 会在程序中添加许多策略类或者策略对象
- 使用策略模式必须了解所有的 strategy, 了解各个 strategy 之间的不同点, 这样才能选择一个合适的 strategy. 此时 strategy 要向客户暴露它的所有实现, 这是违反最少知识原则的.

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
