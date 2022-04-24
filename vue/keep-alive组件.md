# Vue keep-alive 组件

### keep-alive 源码

```javascript
// Vue版本: 2.6.14
/* @flow */

import { isRegExp, remove } from "shared/util";
import { getFirstComponentChild } from "core/vdom/helpers/index";

type CacheEntry = {
  name: ?string,
  tag: ?string,
  componentInstance: Component,
};

type CacheEntryMap = { [key: string]: ?CacheEntry };

// 优先获取组件name选项, 否则获取组件局部注册名称(父组件components选项键值)
function getComponentName(opts: ?VNodeComponentOptions): ?string {
  return opts && (opts.Ctor.options.name || opts.tag);
}

function matches(
  pattern: string | RegExp | Array<string>,
  name: string
): boolean {
  if (Array.isArray(pattern)) {
    return pattern.indexOf(name) > -1;
  } else if (typeof pattern === "string") {
    return pattern.split(",").indexOf(name) > -1;
  } else if (isRegExp(pattern)) {
    return pattern.test(name);
  }
  /* istanbul ignore next */
  return false;
}

function pruneCache(keepAliveInstance: any, filter: Function) {
  const { cache, keys, _vnode } = keepAliveInstance;
  // 遍历cache, 如果缓存没匹配新规则, 则删除缓存
  for (const key in cache) {
    const entry: ?CacheEntry = cache[key];
    if (entry) {
      const name: ?string = entry.name;
      if (name && !filter(name)) {
        pruneCacheEntry(cache, key, keys, _vnode);
      }
    }
  }
}

function pruneCacheEntry(
  cache: CacheEntryMap,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
  const entry: ?CacheEntry = cache[key];
  if (entry && (!current || entry.tag !== current.tag)) {
    entry.componentInstance.$destroy();
  }
  cache[key] = null;
  remove(keys, key);
}

const patternTypes: Array<Function> = [String, RegExp, Array];

// 暴露keep-alive组件对象
export default {
  name: "keep-alive",
  abstract: true, // 抽象组件, 在组件实例建立父子关系时会被忽略

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number],
  },

  methods: {
    cacheVNode() {
      const { cache, keys, vnodeToCache, keyToCache } = this;
      // 缓存vnode
      if (vnodeToCache) {
        const { tag, componentInstance, componentOptions } = vnodeToCache;
        cache[keyToCache] = {
          name: getComponentName(componentOptions),
          tag,
          componentInstance,
        };
        keys.push(keyToCache);
        // 如果超出了最大缓存数, 则删除第一个缓存vnode(最老的缓存vnode)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode);
        }
        this.vnodeToCache = null;
      }
    },
  },

  created() {
    // 缓存vnode
    this.cache = Object.create(null);
    this.keys = [];
  },

  destroyed() {
    // 逐一执行缓存组件的 $destroy
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys);
    }
  },

  mounted() {
    this.cacheVNode();
    // 监测 include 和 exclude 变化
    this.$watch("include", (val) => {
      pruneCache(this, (name) => matches(val, name));
    });
    this.$watch("exclude", (val) => {
      pruneCache(this, (name) => !matches(val, name));
    });
  },

  updated() {
    this.cacheVNode();
  },

  render() {
    const slot = this.$slots.default;
    // 获取第一个子元素的 vnode(只处理第一个子元素)
    const vnode: VNode = getFirstComponentChild(slot);
    const componentOptions: ?VNodeComponentOptions =
      vnode && vnode.componentOptions;
    if (componentOptions) {
      // check pattern
      // 获取组件的 name 或者 tag(组件局部注册的名称)
      const name: ?string = getComponentName(componentOptions);
      const { include, exclude } = this;
      // 判断 include 和 exclude 情况, 不符合缓存条件则直接返回 vnode
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode;
      }

      const { cache, keys } = this;
      // 获取缓存的键值
      const key: ?string =
        vnode.key == null
          ? // same constructor may get registered as different local components
            // so cid alone is not enough (#3269)
            componentOptions.Ctor.cid +
            (componentOptions.tag ? `::${componentOptions.tag}` : "")
          : vnode.key;

      // 命中缓存
      if (cache[key]) {
        // 从缓存中获取组件实例
        vnode.componentInstance = cache[key].componentInstance;
        // 调整key的顺序(放在最后一个)
        // make current key freshest
        remove(keys, key);
        keys.push(key);
      } else {
        // 延迟设置vnode缓存
        // delay setting the cache until update
        this.vnodeToCache = vnode;
        this.keyToCache = key;
      }

      vnode.data.keepAlive = true;
    }
    return vnode || (slot && slot[0]);
  },
};
```

### 渲染过程

keep-alive 中的组件首次渲染跟普通组件渲染没什么区别, 当命中缓存渲染时, 不会触发组件的 `created`, `mounted` 等钩子函数, 而是触发 `acitvated` 钩子(失活对应 `deactivated` 钩子).

渲染过程详细源码分析参见[组件渲染](https://ustbhuangyi.github.io/vue-analysis/v2/extend/keep-alive.html#%E7%BB%84%E4%BB%B6%E6%B8%B2%E6%9F%93)

参考资料

[Vue.js 技术揭秘 keep-alive](https://ustbhuangyi.github.io/vue-analysis/v2/extend/keep-alive.html#%E5%86%85%E7%BD%AE%E7%BB%84%E4%BB%B6)

[Vue 源码系列 keep-alive](https://vue-js.com/learn-vue/BuiltInComponents/keep-alive.html#_1-%E5%89%8D%E8%A8%80)
