# 适配器模式

适配器模式的作用是解决两个软件实体间的接口不兼容的问题. 使用适配器模式之后, 原本由于接口不兼容而不能工作的两个软件实体可以一起工作.

```js
var googleMap = {
  show: function() {
    console.log("开始渲染谷歌地图");
  }
};
var baiduMap = {
  display: function() {
    console.log("开始渲染百度地图");
  }
};

// 适配器, 使地图对象统一提供show接口
var baiduMapAdapter = {
  show: function() {
    return baiduMap.display();
  }
};

var renderMap = function(map) {
  if (map.show instanceof Function) {
    map.show();
  }
};

renderMap(googleMap);
renderMap(baiduMapAdapter);
```

摘自[javascript 设计模式与开发实践](https://book.douban.com/subject/26382780/)
