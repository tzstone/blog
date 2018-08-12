# 组合模式

组合模式就是用小的子对象来构建更大的对象, 而这些小的子对象本身也许是由更小的"孙对象"构成的.

组合模式将对象组合成树形结构, 以表示"`部分-整体`"的层次结构. 组合模式提供了一种遍历树形结构的方案, 通过调用组合对象的`execute`方法, 程序会递归调用组合对象下面的叶对象的`execute`方法(对整个树进行深度优先搜索), 可以很方便地通过一个操作完成多件事情.

通过对象的多态性表现, 使得用户`对单个对象和组合对象的使用具有一致性`(组合对象的子对象可能是单个对象, 也可能是另一个组合对象).

```javascript
var MacroCommand = function() {
  return {
    commandsList: [],
    add: function(command) {
      this.commandsList.push(command);
    },
    execute: function() {
      for (var i = 0, command; (command = this.commandsList[i++]); ) {
        command.execute();
      }
    }
  };
};

var openAcCommand = {
  execute: function() {
    console.log("打开空调");
  }
};

// 宏命令1
var openTvCommand = {
  execute: function() {
    console.log("打开电视");
  }
};
var openSoundCommand = {
  execute: function() {
    console.log("打开音响");
  }
};
var macroCommand1 = MacroCommand();
macroCommand1.add(openTvCommand);
macroCommand1.add(openSoundCommand);

// 宏命令2
var closeDoorCommand = {
  execute: function() {
    console.log("关门");
  }
};
var openPcCommand = {
  execute: function() {
    console.log("开电脑");
  }
};
var openQQCommand = {
  execute: function() {
    console.log("登录QQ");
  }
};
var macroCommand2 = MacroCommand();
macroCommand2.add(closeDoorCommand);
macroCommand2.add(openPcCommand);
macroCommand2.add(openQQCommand);

// 超级命令
var macroCommand = MacroCommand();
macroCommand.add(openAcCommand);
macroCommand.add(macroCommand1);
macroCommand.add(macroCommand2);
macroCommand.execute();
```

## 引用父对象

有时候我们需要在子节点上保持对父节点的引用

```javascript
var Folder = function(name) {
  this.name = name;
  this.parent = null;
  this.files = [];
};

Folder.prototype.add = function(file) {
  file.parent = this; // 设置父对象
  this.files.push(file);
};

Folder.prototype.scan = function(file) {
  console.log("开始扫描文件夹" + this.name);
  for (var i = 0, file, files = this.files; (file = files[i++]); ) {
    file.scan();
  }
};

Folder.prototype.remove = function() {
  // 根节点或树外的游离节点
  if (!this.parent) {
    return;
  }

  // 从父节点中遍历子节点, 找到自己并删除
  for (var files = this.parent.files, l = files.length - 1; l >= 0; l--) {
    var file = files[l];
    if (file === this) {
      files.splice(l, 1);
    }
  }
};

var File = function(name) {
  this.name = name;
  this.parent = null;
};

File.prototype.add = function(file) {
  throw new Error("不能添加在文件下面");
};

File.prototype.scan = function(file) {
  console.log("开始扫描文件" + this.name);
};

File.prototype.remove = function() {
  // 根节点或树外的游离节点
  if (!this.parent) {
    return;
  }

  // 从父节点中遍历子节点, 找到自己并删除
  for (var files = this.parent.files, l = files.length - 1; l >= 0; l--) {
    var file = files[l];
    if (file === this) {
      files.splice(l, 1);
    }
  }
};

var folder = new Folder("学习资料");
var folder1 = new Folder("Javascript");
var file1 = new Folder("深入浅出node");
folder1.add(new File("javascript设计模式与开发实践"));
folder.add(folder1);
folder.add(file1);

folder1.remove();
folder.scan();
```

## 注意的点

- 组合模式不是父子关系

  组合模式是一种`HAS-A`(聚合)的关系, 而不是`IS-A`. 组合对象包含一组叶对象, 但 Left 并不是 Composite 的子类. 组合对象把请求委托给它所包含的所有叶对象, 它们能够合作的关键是拥有相同的接口.

- 对叶对象操作的一致性

  组合模式除了要求组合对象和叶对象拥有相同的接口之外, 对一组叶对象的操作还必须具有一致性.

- 双向映射关系

  如果对象之间的关系并不是严格意义上的层次结构(如某一叶对象同属于两个组合对象), 这种情况必须给父节点和子节点建立双向映射关系(如给父子节点都增加集合来保存对方的引用). 但这种相互间的引用可能会很复杂, 可以通过引入中介者模式来管理这些对象.

- 用职责链模式提高组合模式性能

  在组合模式中, 如果树的结构比较复杂, 节点数量很多, 在遍历树的过程中, 性能方面也许不够理想. 这是可以借助父子对象间形成的天然职责链, 让请求顺着链条从父对象往子对象传递, 或者是反过来从子对象往父对象传递, 直到遇到可以处理该请求的对象为止.

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
