# 装饰者模式

装饰者模式可以动态地给某个对象添加一些额外的职责, 而不会影响从这个类中派生的其他对象.

```js
var Plane = function() {};
Plane.prototype.fire = function() {
  console.log("发射普通子弹");
};
// 装饰类
var MissileDecorator = function(plane) {
  this.plane = plane;
};
MissileDecorator.prototype.fire = function() {
  this.plane.fire(); // 调用plane对象的fire方法
  console.log("发射导弹"); // 执行自身的操作
};

var AtomDecorator = function(plane) {
  this.plane = plane;
};
AtomDecorator.prototype.fire = function() {
  this.plane.fire();
  console.log("发射原子弹");
};

var plane = new Plane();
plane = new MissileDecorator(plane);
plane = new AtomDecorator(plane);
plane.fire();
```

这种给对象动态添加职责的方法, 并没有真正地改动对象本身, 而是`将对象放入另一个对象之中`(实际上, 将装饰者模式称为包装器(`wrapper`)更合适), 这些对象以一条链的方式进行引用, 形成一个聚合对象. 这些对象都拥有相同的接口(`fire`方法), 当请求达到链中的某个对象时, 这个对象会执行自身的操作, 随后把请求转发给链中的下一个对象.

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F-%E9%93%BE.jpeg" align=center width=500/>

因为装饰者对象和它所装饰的对象拥有一致的接口, 所以它们对使用该对象的用户来说是透明的, 被装饰的对象也并不需要了解它曾经被装饰过, 这种透明性使得我们可以递归地嵌套任意多个装饰者对象.

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
