# 代理模式

定义: 代理模式是为一个对象提供一个代用品或占位符, 以便控制对它的访问.

## 保护代理

控制不同权限的对象对目标对象的访问

## 虚拟代理

把一些开销很大的对象, 延迟到真正需要它的时候才去创建

```javascript
// 图片预加载
var myImage = (function() {
  var imgNode = document.createElement('img')
  document.body.appendChild(imgNode)

  return {
    setSrc: function(src) {
      imgNode.src = src
    }
  }
})()

var proxyImage = (function() {
  var img = new Image()
  img.onload = function() {
    myImage.setSrc(this.src)
  }

  return {
    setSrc: function(src) {
      myImage.setSrc('../loading.gif')
      img.src = src
    }
  }
})()

proxyImage.setSrc('http://....a.jpg')
```
