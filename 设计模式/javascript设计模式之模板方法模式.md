# 模板方法模式

模板方法模式由两部分结构组成, 第一部分是抽象父类, 第二部分是具体的实现子类. 通常在抽象父类中封装了子类的算法框架, 包括实现一些公共方法以及封装子类中所有方法的执行顺序. 子类通过继承这个抽象类, 也继承了整个算法结构, 并且可以选择重写父类的方法.

假如我们有一些平行的子类, 各个子类之间有一些相同的行为, 也有一些不同的行为. 如果相同和不同的行为都混合在各个子类的实现中, 说明这些相同的行为会在各个子类中重复出现. 实际上, 相同的方法可以被搬到另一个单一的地方, 模板方法模式就是为解决这个问题而生的. 在模板方法模式中, 子类实现中的相同部分被上移到父类中, 而将不同的部分留待子类来实现.

```javascript
var Beverage = function() {};
// 具体方法, 每个子类都有的同样的具体实现方法
Beverage.prototype.boilWater = function() {
  console.log("把水煮沸");
};
Beverage.prototype.brew = function() {}; // 抽象方法, 应该由子类重写
Beverage.prototype.pourInCup = function() {}; // 抽象方法, 应该由子类重写
Beverage.prototype.addCondiments = function() {}; // 抽象方法, 应该由子类重写
// 模板方法, 封装了子类的算法框架
Beverage.prototype.init = function() {
  this.boilWater();
  this.brew();
  this.pourInCup();
  this.addCondiments();
};

// Coffee子类, 继承Beverage父类
var Coffee = function() {};
Coffee.prototype = new Beverage();
Coffee.prototype.brew = function() {
  console.log("用沸水冲泡咖啡");
};
Coffee.prototype.pourInCup = function() {
  console.log("把咖啡倒进杯子");
};
Coffee.prototype.addCondiments = function() {
  console.log("加糖和牛奶");
};
var coffee = new Coffee();
coffee.init();

// Tea子类, 继承Beverage父类
var Tea = function() {};
Tea.prototype = new Beverage();
Tea.prototype.brew = function() {
  console.log("用沸水浸泡茶叶");
};
Tea.prototype.pourInCup = function() {
  console.log("把茶倒进杯子");
};
Tea.prototype.addCondiments = function() {
  console.log("加柠檬");
};
var tea = new Tea();
tea.init();
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
