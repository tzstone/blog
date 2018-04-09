# forEach详解


`forEach`方法会以升序的顺序对数组的每一项(不包括index属性被删除或未被初始化的项)逐个调用callback方法. `forEach`的第二个参数可以指定callback里面this的指向, 否则callback里this的值为undefined(非严格模式下隐式转换为全局对象, 与普通函数表现一致).

```javascript
var arr = new Array();
arr[2] = 2; // js里数组本质上是hash表, 对arr[2]进行赋值只是添加了一个key-value的映射关系,不会对arr[0], arr[1]进行初始化
arr[4] = undefined;
arr[10] = 10;
arr['a'] = 'a';
delete arr[2];
arr.forEach(function callback(currentValue, index, array) {
    console.log(currentValue); // undefined 10
    console.log(this.num); // 100000
}, {num: 100000})
```


循环遍历数组的长度在第一次调用callback之前就决定了(看结尾的源码就知道了, 会先把length缓存起来), 这意味着在callback里改变数组的长度, 可能会出现一些出乎意料的事情.

```javascript
// t=2, i=1时把arr[2]删掉, 这时arr=[1,2,4,5], 继续执行, 遍历arr[2](=4), arr[3](=5), arr[4], 由于arr[4]没有被初始化,所以不会被遍历到
var arr = [1, 2, 3, 4, 5];
arr.forEach((t, i) => {
    console.log(t, i) // 1 0; 2 1; 4 2; 5 3;
    if (t === 2) {
        arr.splice(i + 1, 1); // delect next
    }
})

// callback里面添加了项目, 不会被遍历到
var arr = [1, 2, 3, 4, 5];
arr.forEach((t, i) => {
    console.log(t, i);
    if (t === 2) {
        arr.push(6);
    }
})
```


有时候我们需要在callback里面根据条件删除一些数组项, 可能会这么写, 看起来好像没啥毛病, 实际上结果是错的, 原因就是splice的时候arr的长度已经被改变了, 但是还是用旧的下标去对arr进行操作. 更好的方式是使用filter来过滤.

```javascript
// 删除arr里与list重复的数
var arr = [1, 2, 3, 4, 5, 6];
var list = [2, 3, 4];

// bad
arr.forEach((t, i) => {
    list.forEach(o => {
        if (t === o) {
            arr.splice(i, 1)
        }
    })
})
// arr = [1, 3, 5, 6]

// good
arr = arr.filter(t => list.indexOf(t) === -1)
```


最后. 贴一下ES5关于forEach实现的源码(源码大法好!)
```javascript
// Production steps of ECMA-262, Edition 5, 15.4.4.18
// Reference: http://es5.github.io/#x15.4.4.18    
  Array.prototype.forEach = function(callback, thisArg) {

    var T, k;

    if (this == null) {
      throw new TypeError(' this is null or not defined');
    }

    var O = Object(this);

    var len = O.length >>> 0;

    if (typeof callback !== "function") {
      throw new TypeError(callback + ' is not a function');
    }

    if (arguments.length > 1) {
      T = thisArg;
    }

    k = 0;

    while (k < len) {
      var kValue;    
      if (k in O) {          
        kValue = O[k];  
        callback.call(T, kValue, k, O); 
      }          
      k++;
    }

  };
```
