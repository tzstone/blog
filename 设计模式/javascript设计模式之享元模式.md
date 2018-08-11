# 享元模式

享元(flyweight)模式是一种用于性能优化的模式, "fly"在这里是苍蝇的意思, 意为蝇量级. 享元模式的核心是运用`共享技术`来有效支持大量细粒度的对象.

享元模式要求将对象的属性划分为内部状态与外部状态(状态在这里通常指属性). 享元模式的目标是尽量减少共享对象的数量. 关于如何划分内部状态和外部状态, 有一些经验:

- 内部状态存储于对象内部
- 内部状态可以被一些对象共享
- 内部状态独立于具体的场景, 通常不会改变
- 外部状态取决于具体的场景, 并根据场景而变化, 外部状态不能被共享

所有内部状态相同的对象都指定为同一个共享的对象, 而外部状态可以从对象身上剥离出来, 并存储在外部. 剥离了外部状态的对象成为共享对象, 外部状态在必要时被传入共享对象来组装成一个完整的对象.

## 适用性

一般来说, 以下情况发生时便可以使用享元模式:

- 一个程序中使用了大量的相似对象
- 由于使用了大量对象, 造成很大的内存开销
- 对象的大多数状态都可以变为外部状态
- 剥离出对象的外部状态之后, 可以用相对较少的共享对象取代大量对象

```javascript
var Model = function(sex) {
  this.sex = sex; // 性别是内部状态
};
Model.prototype.takePhoto = function() {
  console.log("sex=" + this.sex + " underwear=" + this.underwear);
};

var maleModel = new Model("male"),
  femalModel = new Model("female");

for (var i = 0; i <= 50; i++) {
  maleModel.underwear = "underwear" + i; // 内衣是外部状态
  maleModel.takePhoto();
}

for (var j = 0; j <= 50; j++) {
  femalModel.underwear = "underwear" + j;
  femalModel.takePhoto();
}
```

## 享元模式的通用结构

- 当某个共享对象被真正需要时, 才从对象工厂中被创建出来
- 用一个管理器来记录对象相关的外部状态, 使这些外部状态通过某个钩子和共享对象联系起来
  

```javascript
// 共享对象
var Upload = function(uploadType) {
  this.uploadType = uploadType; // 内部状态
};

Upload.prototype.delFile = function(id) {
  // 读取并设置文件的fileSize
  uploadManager.setExternalState(id, this);

  // 文件小于3000k, 直接删除
  if (this.fileSize < 3000) {
    return this.dom.parentNode.removeChild(this.dom);
  }

  // 弹窗确认删除
  if (window.confirm("确定要删除该文件吗?" + this.fileName)) {
    return this.dom.parentNode.removeChild(this.dom);
  }
};

// 对象工厂
var UploadFactory = (function() {
  var createFlyWeightObjs = {};
  return {
    create: function(uploadType) {
      if (createFlyWeightObjs[uploadType]) {
        return createFlyWeightObjs[uploadType];
      }

      return (createFlyWeightObjs[uploadType] = new Upload(uploadType));
    }
  };
})();

// 管理器
var uploadManager = (function() {
  var uploadDatabase = {};
  return {
    add: function(id, uploadType, fileName, fileSize) {
      var flyWeightObj = UploadFactory.create(uploadType);

      var dom = document.createElement("div");
      dom.innerHTML = `<span>文件名称: ${fileName}, 文件大小: ${fileSize}</span>
                      <button class="delFile">删除</button>`;
      dom.querySelector(".delFile").onclick = function() {
        flyWeightObj.delFile(id);
      };
      document.body.appendChild(dom);

      // 保存共享对象的外部状态
      uploadDatabase[id] = {
        fileName: fileName,
        fileSize: fileSize,
        dom: dom
      };

      return flyWeightObj;
    },
    // 动态给共享对象添加外部状态
    setExternalState: function(id, flyWeightObj) {
      var uploadData = uploadDatabase[id];
      for (var i in uploadData) {
        flyWeightObj[i] = uploadData[i];
      }
    }
  };
})();

// 触发上传
var id = 0;
window.startUpload = function(uploadType, files) {
  for (var i = 0, file; (file = files[i]); ) {
    var uploadObj = uploadManager.add(
      ++id,
      uploadType,
      file.fileName,
      file.fileSize
    );
  }
};
```

## 对象池--另一种性能优化方案

对象池维护一个装载`空闲对象`的池子, 如果需要对象的时候, 不是直接 new, 而是转从对象池里获取. 如果对象池里没有空闲对象, 则创建一个新的对象, 当获取出的对象完成它的职责之后, 再进入池子等待被下次获取.

```javascript
var objectPoolFactory = function(createObjFn) {
  var objectPool = []; // 对象池
  return {
    create: function() {
      var obj =
        objectPool.length === 0
          ? createObjFn.apply(this, arguments)
          : objectPool.shift();

      return obj;
    },
    // 对象池回收
    recover: function(obj) {
      objectPool.push(obj);
    }
  };
};

var iframeFactory = objectPoolFactory(function() {
  var iframe = document.createElement("iframe");
  document.body.appendChild(iframe);

  iframe.onload = function() {
    iframe.onload = null; // 防止iframe重复加载的bug
    iframeFactory.recover(iframe); // iframe加载完成后回收节点
  };

  return iframe;
});

var iframe1 = iframeFactory.create();
iframe1.src = "http://baidu.com";

var iframe2 = iframeFactory.create();
iframe2.src = "http://QQ.com";

setTimeout(function() {
  var iframe3 = iframeFactory.create();
  iframe3.src = "http://163.com";
}, 30000);
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
