# 组合模式

组合模式就是用小的子对象来构建更大的对象, 而这些小的子对象本身也许是由更小的"孙对象"构成的.

组合模式将对象组合成树形结构, 以表示"部分-整体"的层次结构. 通过对象的多态性表现, 使得用户`对单个对象和组合对象的使用具有一致性`(组合对象的子对象可能是单个对象, 也可能是另一个组合对象).

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
