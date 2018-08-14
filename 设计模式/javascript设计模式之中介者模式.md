# 中介者模式

中介者模式的作用是解除对象与对象之间的紧耦合关系. 增加一个中介者对象后, 所有的相关对象都通过中介者对象来通信, 而不是相互引用, 所以当一个对象发生改变时, 只需要通知中介者对象即可. 中介者模式使网状的多对多关系变成了相对简单的一对多关系.

before:

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/%E4%B8%AD%E4%BB%8B%E8%80%85%E6%A8%A1%E5%BC%8F-before.jpeg" align=center width=500/>

after:

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/%E4%B8%AD%E4%BB%8B%E8%80%85%E6%A8%A1%E5%BC%8F-after.jpeg" align=center width=500/>

## 缺点

系统中会新增一个中介者对象, 因为对象之间交互的复杂性, 转移成了中介者对象的复杂性, 使得中介者对象经常是巨大的. 中介者对象自身往往就是一个难以维护的对象.

```javascript
function Player(name, teamColor) {
  this.name = name;
  this.teamColor = teamColor;
  this.state = "alive"; // 玩家生存状态
}
Player.prototype.win = function() {
  console.log(this.name + " won ");
};
Player.prototype.lose = function() {
  console.log(this.name + " lose ");
};
Player.prototype.die = function() {
  this.state = "dead";
  playerDirector.ReceiveMessage("playerDead", this); // 向中介者发消息, 玩家死亡
};
Player.prototype.remove = function() {
  playerDirector.ReceiveMessage("removePlayer", this); // 向中介者发消息, 移除一个玩家
};
Player.prototype.changeTeam = function(color) {
  playerDirector.ReceiveMessage("changeTeam", this, color); // 向中介者发消息, 玩家换队
};

// 创建玩家的工厂函数
var playerFactory = function(name, teamColor) {
  var newPlayer = new Player(name, teamColor); // 创建一个新玩家对象
  playerDirector.ReceiveMessage("addPlayer", newPlayer); // 向中介者发消息, 新增玩家
  return newPlayer;
};

// 中介者
var playerDirector = (function() {
  var players = {}, // 保存所有玩家
    operations = {}; // 中介者可以执行的操作

  operations.addPlayer = function(player) {
    var teamColor = player.teamColor;
    players[teamColor] = players[teamColor] || [];
    players[teamColor].push(player);
  };
  operations.removePlayer = function(player) {
    var teamColor = player.teamColor,
      teamPlayers = players[teamColor] || [];

    // 倒序遍历删除, 避免正序遍历删除带来的数组乱序
    for (var i = teamPlayers.length - 1; i >= 0; i--) {
      if (teamPlayers[i] === player) {
        teamPlayers.splice(i, 1);
      }
    }
  };
  operations.changeTeam = function(player, newTeamColor) {
    operations.removePlayer(player);
    player.teamColor = newTeamColor;
    operations.addPlayer(player);
  };
  operations.playerDead = function(player) {
    var teamColor = player.teamColor,
      teamPlayers = players[teamColor];

    var all_dead = true;
    for (var i = 0, player; (player = teamPlayers[i++]); ) {
      if (player.state !== "dead") {
        all_dead = false;
        break;
      }
    }
    // 全部死亡
    if (all_dead === true) {
      for (var i = 0, player; (player = teamPlayers[i++]); ) {
        player.lose(); // 本队所有玩家lose
      }
      for (var color in players) {
        if (color !== teamColor) {
          var teamPlayers = players[color]; // 其他队伍的玩家
          for (var i = 0, player; (player = teamPlayers[i++]); ) {
            player.win();
          }
        }
      }
    }
  };

  var ReceiveMessage = function() {
    var message = Array.prototype.shift.call(arguments); // 第一个参数为消息名称
    operations[message].apply(this, arguments);
  };

  return {
    ReceiveMessage: ReceiveMessage
  };
})();

var player1 = playerFactory("小A", "red");
var player2 = playerFactory("小B", "red");
var player3 = playerFactory("小C", "blue");
var player4 = playerFactory("小D", "blue");

player1.die();
player2.die();
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
