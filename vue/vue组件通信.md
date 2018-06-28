# 父子组件

## 父组件向子组件通信

### 1. 通过[props](https://cn.vuejs.org/v2/guide/components.html)向子组件传递数据

父组件

```javascript
<template>
  <div>
    <child :msg='msg'></child>
  </div>
</template>

<script>
  import child from './child.vue'
  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    components: {
      child
    }
  }
</script>
```

子组件

```javascript
<template>
  <div>{{msg}}</div>
</template>

<script>
  export default {
    props: ['msg']
  }
</script>
```

### 2. 使用[vm.$children](https://cn.vuejs.org/v2/api/#vm-children)访问当前实例的直接子组件列表

注意: $children 不保证顺序, 也不是响应式的(如无法 watch '$children'的变化)

父组件

```javascript
<template>
  <div>
    <child1 :msg='msg'></child1>
    <child2 :msg='msg'></child2>
  </div>
</template>

<script>
  import child1 from './child1.vue'
  import child2 from './child2.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    mounted() {
      console.log(this.$children) // [VueComponent(child1), VueComponent(child2)]
      // 通过VueComponent的$options._componentTag可找到对应的component, 
      // 比如注册child1组件时写成components: {'child1-1': child1}, 那么_componentTag就是'child1-1'
    },
    components: {
      child1,
      child2
    }
  }
</script>
```

子组件

```javascript
// child1.vue
<template>
  <div>{{msg}}</div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child1'
      }
    },
    props: ['msg']
  }
</script>


// child2.vue
<template>
  <div>{{msg}}</div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child2'
      }
    },
    props: ['msg']
  }
</script>
```

### 3. 使用[ref](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E5%AD%90%E7%BB%84%E4%BB%B6%E5%AE%9E%E4%BE%8B%E6%88%96%E5%AD%90%E5%85%83%E7%B4%A0)特性访问子组件

注意: $refs 只会在组件渲染完成之后生效，并且它们不是响应式的。这只意味着一个直接的子组件封装的“逃生舱”——你应该避免在模板或计算属性中访问 $refs。

父组件

```javascript
<template>
  <div>
    <!-- 使用ref特性为子组件赋予一个ID引用 -->
    <child ref='child' :msg='msg'></child>
  </div>
</template>

<script>
  import child from './child.vue'
  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    mounted() {
      console.log(this.$refs.child) // VueComponent(child)
    },
    components: {
      child
    }
  }
</script>
```

子组件

```javascript
<template>
  <div>{{msg}}</div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child'
      }
    },
    props: ['msg']
  }
</script>
```

## 子组件向父组件通信

### 1. 通过 [$emit](https://cn.vuejs.org/v2/guide/components.html#%E9%80%9A%E8%BF%87%E4%BA%8B%E4%BB%B6%E5%90%91%E7%88%B6%E7%BA%A7%E7%BB%84%E4%BB%B6%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF) 方法向父组件触发事件

 父组件

```javascript
<template>
  <div>
    <child @clickBtn='clickBtn' :msg='msg'></child>
  </div>
</template>

<script>
  import child from './child.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    methods: {
      clickBtn(val) {
        console.log('event from', val)
      }
    },
    components: {
      child
    }
  }
</script>
```

子组件

```javascript
<template>
  <div>
    <span>{{msg}}</span>
    <button @click="handleEmit">emit</button>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child'
      }
    },
    props: ['msg'],
    methods: {
      handleEmit() {
        this.$emit('clickBtn', this.name)
      }
    }
  }
</script>
```

### 2. 通过[$parent](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E7%88%B6%E7%BA%A7%E7%BB%84%E4%BB%B6%E5%AE%9E%E4%BE%8B)访问直接父组件实例

注意: 可能会让程序更难调试和理解, 假如父组件数据发生了变更, 很难找出是由哪个子组件触发的.

类似$parent, 可通过[$root](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E6%A0%B9%E5%AE%9E%E4%BE%8B)访问根实例

 父组件

```javascript
<template>
  <div>
    <child :msg='msg'></child>
  </div>
</template>

<script>
  import child from './child.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    components: {
      child
    }
  }
</script>
```

子组件

```javascript
<template>
  <div>
    <span>{{msg}}</span>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child'
      }
    },
    props: ['msg'],
    mounted() {
      console.log(this.$parent) // VueComponent(father)
    }
  }
</script>
```

### 3. 通过[依赖注入](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)

$parent 只能访问直接父组件, 而依赖注入可以让某个父组件的数据/方法被任何后代组件访问

注意: 依赖注入会使重构变得更加困难,  同时所提供的属性是非响应式的.

父组件

```javascript
<template>
  <div>
    <child :msg='msg'></child>
  </div>
</template>

<script>
  import child from './child.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    provide: function () {
      return {
        getMsg: this.getMsg
      }
    },
    methods: {
      getMsg(name) {
        console.log('invoke getMsg from', name)
      }
    },
    components: {
      child
    }
  }
</script>
```

子组件

```javascript
<template>
  <div>
    <span>{{msg}}</span>
    <grandson></grandson>
  </div>
</template>

<script>
  import grandson from './grandson.vue'
  export default {
    data() {
      return {
        name: 'child'
      }
    },
    inject: ['getMsg'],
    props: ['msg'],
    mounted() {
      this.getMsg(this.name) // invoke getMsg from child
    },
    components: {
      grandson
    }
  }
</script>
```

孙子组件

```javascript
<template>
  <div>
    <span>grandson</span>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'grandson'
      }
    },
    inject: ['getMsg'],
    mounted() {
      this.getMsg(this.name) // invoke getMsg from child
    }
  }
</script>
```

### 4. 通过直接修改 props 参数(不推荐)

当父组件通过 props 传递一个引用类型数据时, 子组件可以修改传入的数据来更改父组件中对应的数据, 但这样会增加父子组件的耦合程度, 出现问题也不好查找. 如果担心子组件不小心更改了父组件的数据, 一般会用 var proxyData = JSON.parse(JSON.stringify(data))的方式来让数据指向不同的内存地址.

# 兄弟组件

### 1. 通过父组件访问

父组件

```javascript
<template>
  <div>
    <child1 ref='child1' :msg='msg'></child1>
    <child2 ref='child2' :msg='msg'></child2>
  </div>
</template>

<script>
  import child1 from './child1.vue'
  import child2 from './child2.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    components: {
      child1,
      child2
    }
  }
</script>
```

子组件

```javascript
// child1.vue
<template>
  <div>
    <span>{{msg}}</span>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        name: 'child1'
      }
    },
    props: ['msg'],
    mounted() {
      console.log(this.$parent.$refs.child2) // VueComponent(child2)
    }
  }
</script>

// child2.vue
<template>
  <div>{{msg}}</div>
</template>

<script>
  export default {
    props: ['msg']
  }
</script>
```

# 其他

以下方法均适用任何类型的组件

### 1. [事件总线](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E7%A8%8B%E5%BA%8F%E5%8C%96%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%BE%A6%E5%90%AC%E5%99%A8)

适合所有类型组件的消息传递, 这里以兄弟组件为例

事件总线

```javascript
// event.js
// 新建一个vue实例作为事件总线
import Vue from 'vue'

var event = new Vue()

export default event
```

父组件

```javascript
<template>
  <div>
    <child1 :msg='msg'></child1>
    <child2 :msg='msg'></child2>
  </div>
</template>

<script>
  import child1 from './child1.vue'
  import child2 from './child2.vue'

  export default {
    data() {
      return {
        msg: 'father'
      }
    },
    components: {
      child1,
      child2
    }
  }
</script>
```

子组件

```javascript
// child1.vue
<template>
  <div>
    <span>{{msg}}</span>
  </div>
</template>

<script>
  import event from './event.js'
  export default {
    data() {
      return {
        name: 'child1'
      }
    },
    props: ['msg'],
    mounted() {
      // 监听clickBtn事件
      event.$on('clickBtn', (name) => {
        console.log('trigger click event from', name)
      })
    }
  }
</script>

// child2.vue
<template>
  <div>
    <span>{{msg}}</span>
    <button @click="handleClick">click me</button>
  </div>
</template>

<script>
  import event from './event.js'
  export default {
    data() {
      return {
        name: 'child2'
      }
    },
    props: ['msg'],
    methods: {
      handleClick() {
        // 触发clickBtn事件
        event.$emit('clickBtn', this.name)
      }
    }
  }
</script>
```

### 2. 使用[vuex](https://cn.vuejs.org/v2/guide/state-management.html)进行状态管理

### 3. 使用 localStorage/sessionStorage 进行全局数据存储/访问
