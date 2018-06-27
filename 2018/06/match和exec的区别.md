str.match()方法和 RegExp.exec()方法同样可以用来返回匹配文本数据, 但也有一些差异, 记录一下.

## 非全局模式

非全局模式下 str.match() 和 RegExp.exec()返回结果是一样的.

### str.match()

```javascript
var result = [];

var str =
  "<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>";

var reg = /<li>(.*?)<\/li>/;

result = str.match(reg);
```

结果:
![](https://raw.githubusercontent.com/tzstone/MarkdownPhotos/master/match-%E9%9D%9E%E5%85%A8%E5%B1%80.png)

可以看到返回了一个数组, 数组第一项是匹配文本, 其余项是子表达式的匹配文本(第二项是分组 1 的匹配文本, 第三项是分组 2 的匹配文本. 以此类推).
如果没有找到任何匹配文本, match 方法会返回 null.
这个数组还有另外几个属性:

- input 是对进行匹配的原始字符串的引用
- index 表示匹配文本的起始字符在原始字符串中的位置

### RegExp.exec()

```javascript
var result = [];

var str =
  "<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>";

var reg = /<li>(.*?)<\/li>/;

result = reg.exec(str);
```

结果:
![](https://github.com/tzstone/MarkdownPhotos/blob/master/exec-%E9%9D%9E%E5%85%A8%E5%B1%80.png)

可以看到跟 str.match()的返回结果是完全一样的
如果没有找到任何匹配文本, exec 方法会返回 null.

## 全局模式

### str.match()

```javascript
var result = [];

var str =
  "<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>";

var reg = /<li>(.*?)<\/li>/g;

result = str.match(reg);
```

结果:
![](https://github.com/tzstone/MarkdownPhotos/blob/master/match-%E5%85%A8%E5%B1%80.png)

可以看到, match 方法同样返回一个数组, 数组里存放匹配到的所有子字符串, 数组没有包含 index 和 input 属性.

### RegExp.exec()

```javascript
var result = [];

var str =
  "<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>";

var reg = /<li>(.*?)<\/li>/g;

result = reg.exec(str);
```

结果:
![](https://github.com/tzstone/MarkdownPhotos/blob/master/exec-%E5%85%A8%E5%B1%80.png)

全局模式下, exec()方法会在 RegExpObject 的 lastIndex 属性指定的字符处开始检索字符串. 当 exec()找到了与表达式匹配的文本后, 会将 RegExpObject 的 lastIndex 属性设置为匹配文本的最后一个字符的下一个位置(非全局模式下, 则不会修改 lastIndex). 可以通过反复调用 exec()方法来遍历字符串中的所有匹配文本. 当 exec()没有找到匹配文本的时候, 会返回 null, 同时把 RegExpObject 的 lastIndex 属性重置为 0.

注意: 如果在一个字符串中完成了一次模式匹配之后要开始检索新的字符串，就必须手动地把 lastIndex 属性重置为 0.

举个栗子:

```javascript
var result = [];

var str =
  "<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>";

var reg = /<li>(.*?)<\/li>/g;

result = reg.exec(str);

var tmp = reg.exec("<li>test</li>"); // tmp为null, 因为在上一条语句进行模式匹配时, reg.lastIndex已经修改了, 必须先手动设置为0: reg.lastIndex=0
```

### 总结

非全局模式下, match()方法和 exec()方法没有区别; 全局模式下, match()方法只返回匹配的子字符串, 而通过循环调用 exec()方法可以返回更完整的匹配信息(包括匹配的子字符串和子表达式的匹配文本).
