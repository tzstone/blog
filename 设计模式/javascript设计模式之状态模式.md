# 状态模式

状态模式的关键是区分事物内部的状态, 事物内部状态的改变往往会带来事物的行为改变.

把事物的每个状态都封装成单独的类, 跟此种状态有关的行为都被封装在这个类的内部, 触发状态转移时, 把请求委托给当前的状态对象.

```js
// 状态类, 封装状态相关的行为
var OffLightState = function(light) {
  this.light = light;
};
// 状态对象持有对light的引用, 以便调用light的方法或操作其对象
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

// Light类(上下文 Context)持有状态对象的引用, 以便把请求委托给状态对象
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

复杂切换条件的状态模式

```js
// 文件上传插件
var plugin = (function() {
  var plugin = document.createElement("embed");
  plugin.style.display = "none";
  plugin.type = "application/txftn-webkit";

  plugin.sign = function() {
    console.log("开始文件扫描");
  };
  plugin.pause = function() {
    console.log("暂停文件上传");
  };
  plugin.uploading = function() {
    console.log("开始文件上传");
  };
  plugin.del = function() {
    console.log("删除文件上传");
  };
  plugin.done = function() {
    console.log("文件上传成功");
  };

  document.body.appendChild(plugin);
  return plugin;
})();

var Upload = function(fileName) {
  this.plugin = plugin;
  this.fileName = fileName;
  this.button1 = null;
  this.button2 = null;

  // 持有状态对象
  this.signState = new SignState(this);
  this.uploadingState = new UploadingState(this);
  this.pauseState = new PauseState(this);
  this.doneState = new DoneState(this);
  this.errorState = new ErrorState(this);

  this.currState = this.signState; // 设置当前状态
};
Upload.prototype.init = function() {
  this.dom = document.createElement("div");
  this.dom.innerHTML = `
    <span>文件名称: ${this.fileName}</span>
    <button data-action="button1">扫描中</button>
    <button data-action="button2">删除</button>`;

  document.body.appendChild(this.dom);

  this.button1 = this.dom.querySelector('[data-action="button1"]');
  this.button2 = this.dom.querySelector('[data-action="button2"]');
  this.bindEvent();
};
Upload.prototype.bindEvent = function() {
  var self = this;
  // 请求委托给状态对象
  this.button1.onclick = function() {
    self.currState.clickHandler1();
  };
  this.button2.onclick = function() {
    self.currState.clickHandler2();
  };
};
/*
  状态对应的逻辑行为放在Upload类中, 而不是状态类中,
  这是因为状态类中状态的切换不是单一的, 一个状态类可以切换到多个其他状态,
  把状态对应的逻辑行为写在Upload类中有利于代码复用和维护
*/
Upload.prototype.sign = function() {
  this.plugin.sign();
  this.currState = this.signState;
};
Upload.prototype.uploading = function() {
  this.button1.innerHTML = "正在上传, 点击暂停";
  this.plugin.uploading();
  this.currState = this.uploadingState;
};
Upload.prototype.pause = function() {
  this.button1.innerHTML = "已暂停, 点击继续上传";
  this.plugin.pause();
  this.currState = this.pauseState;
};
Upload.prototype.done = function() {
  this.button1.innerHTML = "上传完成";
  this.plugin.done();
  this.currState = this.doneState;
};
Upload.prototype.error = function() {
  this.button1.innerHTML = "上传失败";
  this.currState = this.errorState;
};
Upload.prototype.del = function() {
  this.plugin.del();
  this.dom.parentNode.removeChild(this.dom);
};

// 状态工厂
var StateFactory = (function() {
  // 抽象父类, 避免子类没有定义clickHandler方法
  var State = function() {};
  State.prototype.clickHandler1 = function() {
    throw new Error("子类必须重写父类的clickHandler1方法");
  };
  State.prototype.clickHandler2 = function() {
    throw new Error("子类必须重写父类的clickHandler2方法");
  };

  return function(param) {
    var F = function(uploadObj) {
      this.uploadObj = uploadObj; // 持有对Upload的引用
    };
    F.prototype = new State(); // 继承抽象父类
    for (var i in param) {
      F.prototype[i] = param[i];
    }
    return F;
  };
})();

var SignState = StateFactory({
  clickHandler1: function() {
    console.log("扫描中, 点击无效...");
  },
  clickHandler2: function() {
    console.log("文件正在上传中, 不能删除");
  }
});

var UploadingState = StateFactory({
  clickHandler1: function() {
    this.uploadObj.pause();
  },
  clickHandler2: function() {
    console.log("文件正在上传中, 不能删除");
  }
});
var PauseState = StateFactory({
  clickHandler1: function() {
    this.uploadObj.uploading();
  },
  clickHandler2: function() {
    this.uploadObj.del();
  }
});
var DoneState = StateFactory({
  clickHandler1: function() {
    console.log("文件已上传完成, 点击无效");
  },
  clickHandler2: function() {
    this.uploadObj.del();
  }
});
var ErrorState = StateFactory({
  clickHandler1: function() {
    console.log("文件上传失败, 点击无效");
  },
  clickHandler2: function() {
    this.uploadObj.del();
  }
});

var uploadObj = new Upload("javascript");
uploadObj.init();

window.external.upload = function(state) {
  uploadObj[state]();
};
window.external.upload("sign");
setTimeout(() => {
  window.external.upload("uploading");
}, 1000);
setTimeout(() => {
  window.external.upload("done");
}, 5000);
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
