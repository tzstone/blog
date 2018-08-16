# 状态模式

状态模式的关键是区分事物内部的状态, 事物内部状态的改变往往会带来事物的行为改变.

把事物的每个状态都封装成单独的类, 跟此种状态有关的行为都被封装在这个类的内部, 触发状态转移时, 把请求委托给当前的状态对象.

```js
// 状态类, 封装状态相关的行为
var OffLightState = function(light) {
  this.light = light;
};
OffLightState.prototype.buttonWasPressed = function() {
  console.log("弱光");
  this.light.setState(this.light.weakLightState); // 切换到weakLightState
};
var WeakLightState = function(light) {
  this.light = light;
};
WeakLightState.prototype.buttonWasPressed = function() {
  console.log("强光");
  this.light.setState(this.light.strongLightState); // 切换到strongLightState
};
var StrongLightState = function(light) {
  this.light = light;
};
StrongLightState.prototype.buttonWasPressed = function() {
  console.log("关灯");
  this.light.setState(this.light.offLightState); // 切换到offLightState
};

var Light = function() {
  this.offLightState = new OffLightState(this);
  this.weakLightState = new WeakLightState(this);
  this.strongLightState = new StrongLightState(this);
  this.button = null;
};
Light.prototype.init = function() {
  var button = document.createElement("button"),
    self = this;

  this.button = document.body.appendChild(button);
  this.button.innerHTML = "开关";
  this.currState = this.offLightState; // 设置当前状态
  this.button.onclick = function() {
    self.currState.buttonWasPressed();
  };
};
Light.prototype.setState = function(newState) {
  this.currState = newState;
};
var light = new Light();
light.init();
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
