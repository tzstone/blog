String 的 match 方法和 RegExp 的 exec 方法  同样可以用来  返回匹配文本数据, 但也有一些差异, 记录一下.

## 非全局模式

非全局模式下 str.match 和 RegExp.exec()返回结果是一样的.

str.match

```javascript
var result = []

var str = '<p>打发的</p ><ol><li>打发大师傅</li></ol><ul><li>3&nbsp;<br></li><li>打发</li></ul>'

var reg =  /<li>(.*?)<\/li>/

result = str.match(reg)

![](https://raw.githubusercontent.com/tzstone/MarkdownPhotos/master/match-%E9%9D%9E%E5%85%A8%E5%B1%80.png)
```

## 全局模式
