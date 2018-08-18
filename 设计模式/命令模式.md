# 命令模式

命令模式中的命令(command)指的是一个执行某些特定事情的指令. 命令模式最常见的应用场景是: 有时候需要向某些对象发送请求, 但是并不知道请求的接受者是谁, 也不知道被请求的操作是什么. 此时希望用一种松耦合的方式来设计程序, 使得请求发送者和请求接收者能够消除彼此之间的耦合关系.

命令模式的由来, 其实是回调函数的一个面向对象的替代品.

## 实现

```javascript
var MenuBar = {
  refresh: function() {
    console.log("刷新页面");
  }
};

var RefreshMenuBarCommand = function(receiver) {
  return {
    execute: function() {
      receiver.refresh();
    }
  };
};

var setCommand = function(button, command) {
  button.click = function() {
    command.execute();
  };
};

var refreshMenuBarCommand = RefreshMenuBarCommand(MenuBar);
setCommand(button1, refreshMenuBarCommand);
```

## 撤销和重做

保存一个执行的命令的历史列表, 对于可逆命令, 可以通过倒序循环依次执行这些命令的 undo 操作进行命令撤销. 对于不可逆命令, 可以清除结果, 通过重新执行命令进行重做.

```javascript
var Ryu = {
  attack: function() {
    console.log("攻击");
  },
  defense: function() {
    console.log("防御");
  },
  jump: function() {
    console.log("跳跃");
  },
  crouch: function() {
    console.log("蹲下");
  }
};

var makeCommand = function(receiver, state) {
  return function() {
    receiver[state]();
  };
};

var commands = {
  "119": "jump", // W
  "115": "crouch", // S
  "97": "defense", // A
  "100": "attack" // D
};

var commandStack = []; // 保存命令的堆栈
document.onkeypress = function(ev) {
  var keyCode = ev.keyCode,
    command = makeCommand(Ryu, commands[keyCode]);
  if (command) {
    command();
    commandStack.push(command); // 保存刚刚执行过的命令
  }
};

document.getElementById("replay").onclick = function() {
  var command;
  while ((command = commandStack.shift())) {
    // 从堆栈里依次取出命令并执行
    command();
  }
};
```

## 宏命令

宏命令是一组命令的集合, 通过执行宏命令的方式, 可以一次执行一批命令.

```javascript
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

var macroCommand = MacroCommand();
macroCommand.add(closeDoorCommand);
macroCommand.add(openPcCommand);
macroCommand.add(openQQCommand);
macroCommand.execute();
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
