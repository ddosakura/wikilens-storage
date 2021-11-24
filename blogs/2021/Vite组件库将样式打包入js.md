# Vite 组件库将样式打包入 js

## 背景

在之前基于 rollup 的组件库中，[rollup-plugin-vue](https://rollup-plugin-vue.vuejs.org/options.html#css) 插件有一个 css 选项，可以决定样式是提取为单独的文件还是打包入 js。对于组件库来说，为了使用方便以及方便 [[Tree Shaking]]，这是一个很有用的选项。

在将组件库升级到 vue2 与 vue3 兼容的同时，我们采用了 vite 作为新的开发与构建工具。然而 vite 将 [CSS 代码分割](https://cn.vitejs.dev/guide/features.html#css-code-splitting) 的工作进行了高层的封装——根据 chunk 进行 css 的拆分，同时在库模式中默认不进行拆分（支持 es 下的拆分，不支持 umd 下的拆分）。

### 为什么 vite 不提供相关选项

根据 [vite#1579](https://github.com/vitejs/vite/issues/1579#issuecomment-763295757) 中尤大的说法，在 js 中进行样式插入的操作，假定了代码当前在 Dom 环境下，可能会导致不兼容 SSR，目前没有支持计划。

## rollup 相关插件的实现

vite 基于 rollup 进行打包，理论上来讲，rollup 能做的事，vite 也能做。

### rollup-plugin-vue 是如何做的

我们先来看看 [rollup-plugin-vue](https://github1s.com/vuejs/rollup-plugin-vue/blob/2.2/src/style.js) 是怎么做的，先看一段其生成的代码：

```js
const __vue_inject_styles__ = function (inject) {
  if (!inject) return
  inject("data-v-<hash>", { source: "这里是样式", map: undefined, media: undefined });
};
const __component__ = /*#__PURE__*/normalizeComponent(
  { render: __vue_render__, staticRenderFns: __vue_staticRenderFns__ },
  __vue_inject_styles__,
  __vue_script__,
  __vue_scope_id__,
  __vue_is_functional_template__,
  __vue_module_identifier__,
  false,
  browser,
  undefined,
  undefined
);
export default /*#__PURE__*/(function () { return __component__.exports })()
```

不巧，这里的逻辑和 vue2 的模板编译代码是耦合的，一个 vue3 的组件编译后与其天差地远，并且似乎没提供类似的能力：

```js
const _sfc_main$2 = defineComponent( /* 组件定义*/ );
const _sfc_render$2 = // vu3 渲染函数
export default /* @__PURE__ */ _export_sfc(_sfc_main$2, [["render", _sfc_render$2], ["__scopeId", "data-v-hash"]]);
```

### rollup-plugin-postcss 是如何做的

还有另一个 rollup 插件与 css 在 js 中的使用相关（`import './***.css';`）： 

```js
postcss({
  extensions: ['.css', '.postcss'],
  inject: {
    insertAt: 'top',
  },
}),
```

其生成的产物为：

```js
// styleInject 是一个第三方库
// import styleInject from 'style-inject'

var css = "这里是样式";
styleInject(css,{"insertAt":"top"});
```

此方案和框架版本无关，比较通用。

## 实现 vite 插件

### 寻找插件的执行时机

对于一个和 vue SFC 代码转换强相关的插件，插件的执行时机非常重要。

1. Alias
2. 带有 enforce: 'pre' 的用户插件
3. Vite 核心插件
4. 没有 enforce 值的用户插件
5. Vite 构建用的插件
6. 带有 enforce: 'post' 的用户插件
7. Vite 后置构建插件（最小化，manifest，报告）

我们首先梳理一下在几个核心位置 `*.vue` 代码的状态：

+ `2` 时，表现为 `SFC` 源码
+ `4` 时置于 SFC 插件前，表现为 JS 代码加虚拟样式引入（还未加入 scoped）
+ `4` 时置于 SFC 插件后，表现为 JS 代码加虚拟样式引入（样式内容已正确处理）
+ `6` 时，表现为 JS 代码加虚拟样式引入，但经过调试，此时 CSS 中的内容为 `export default ''`，说明 vite 在 `5` 中抽离了所有样式内容

### 初步实现

首先的想法是，我们先在 `4` 时截留 `SFC` 的虚拟样式内容，通过 id 缓存下来，并在 `6` 中注入样式，核心代码如下：

```ts
{
  async transform(src, id) {
    if (!/packages.*\.vue/.test(id)) return
    if (/vue&type=style/.test(id)) {
      // 缓存样式
      cache[id] = src
      // 清除虚拟样式的内容
      return ''
    }
    return src
  },
}
```

```ts
  {
    enforce: 'post',
    async transform(src, id) {
      if (!/packages.*\.vue/.test(id)) return
      if (/vue&type=style/.test(id)) {
        return `export default \`${cache[id]}\``
      }
      // 插入样式
    },
  }
```

以上方案能实现注入样式的效果，同时单独的 `style.css` 也消失了，可喜可贺！

### 简化并优化实现

在之前的实现中，我们处理了 SFC 中的样式，但并不处理非 SFC 中的样式：

```js
import '@/style/global.css'
```

但这也给了我们思路，vite 以及处理了样式直接 import 的转换！

先看一个 SFC 组件中虚拟样式的导入：

```js
import ".....view.vue?vue&type=style&index=0&scoped=true&lang.less"`
```

我们可以知道，这是一个样式（`type=style`）；SFC 中第一个样式块（`index=0`）；使用了 `scoped`（scoped=true）；使用了 Less （`lang.less`）。

这里有一个奇怪的地方，为啥是 `lang.less` 而不是和其它一样的 `lang=less`？

说明这个 `.less` 在最后可不是瞎写的，它是在告诉 vite，虽然我是虚拟的，但我是样式！

既然如此事情就简单了，我们把所有样式统一按照直接 import 的方式来处理，事情就解决了，也就是变为：

```js
import styleInject from 'style-inject'
import css from '.....view.vue?vue&type=style&index=0&scoped=true&lang.less'
styleInject(css, {insertAt:"top"})
```

插件的核心代码如下：

```ts
{
  async transform(src, id) {
    if (!/packages.*\.vue/.test(id)) return
    if (/vue&type=style/.test(id)) return
    let firstInject = true
    const code = src.replace(/import "(.*\.(css|less|scss|sass))"/mg, (_$0, $1) => {
      const [name] = $1.split('/').slice(-1)
      const cssid = name.replace(/[?&=\.]/g, '_')
      const code = `${ firstInject ? `import styleInject from 'style-inject';` : '' }
import ${cssid} from '${$1}';
styleInject(${cssid}, {insertAt:"top"});`
      firstInject = false
      return code
    });
    return code
  },
})
```

美中不足的是，该方案虽然可以处理所有样式的引入，但依旧会生成 `stlye.css`（由于未清除样式文件的内容，vite 对其进行了提取），这里我们忽略即可。

### 按需加载

经过之前的转换，我们成功在代码中插入了样式。可是，这并不是按需加载的，也无法 [[Tree Shaking]]！

但假如无法做到这一点，我们将样式加入 js 的就成了画蛇添足，我们需要在真正使用到组件的时候，再执行 `styleInject`。

所以，第一步，我们先将需要插入的代码拆分为两部分：

```ts
// 插在最前面
const imports = [`import styleInject from 'style-inject'`]
// 按需加载
const injects = [] as string[]
const middleCode = src.replace(/import "(.*\.(css|less|scss|sass))"/mg, (_$0, $1) => {
  const [name] = $1.split('/').slice(-1)
  const cssid = name.replace(/[?&=\.]/g, '_')
  imports.push(`import ${cssid} from '${$1}'`)
  injects.push(`styleInject(${cssid}, {insertAt:"top"})`)
  return ''
});
```

可我们在什么地方按需加载呢？

#### 在 setup 中

首先想到的是写入 `setup` （因为仓库中所由组件都迁移到了组合式 api 的实现方式）。

但是一则写入其中需要保证多个组件实例只会调用一次插入样式，实现相对麻烦，二则并没有较好的在不进行 AST 解析的情况下往函数里插代码的思路。

#### 改写 vue2&vue3 的组件包装函数

于是我换了个思路，说到 [[Tree Shaking]]，那就不得不提纯函数。SFC 转换后的代码中有 `/*#__PURE__*/` 标记吗？必然有（通过开发者写的普通对象来创建组件对象的过程必然只需执行一次且需要支持组件的 [[Tree Shaking]]）！

```js
// vue2
export default /*#__PURE__*/(function () { return __component__.exports })()
// vue3
export default /* @__PURE__ */ _export_sfc(_sfc_main$2, [["render", _sfc_render$2], ["__scopeId", "data-v-hash"]]);
```

+ 对于 vue2，直接往这个 [[IIFE]] 里写即可
+ 对于 vue3，则复杂一些，需要把 `_export_sfc` 包装成一个 [[IIFE]]：

```ts
code.replace('_export_sfc(', `((...args) => (${injects.join(',')},_export_sfc(...args)))(`)
```

### 样式的禁用

有的时候，使用方并不想引入样式。可以通过定义一个 window 下的变量来实现这一能力。

```ts
const inject = `typeof window.DISABLE_STYLE_INJECT === 'undefined' && (${injects.join(',')})`
```

使用 window 下的变量来进行判断有两点好处。在通过 cdn 使用时（by umd），只需要在使用前设置该变量即可生效：

```ts
window.DISABLE_STYLE_INJECT = true;
```

在 nodejs 中使用时（by esm），我们可以替换该值来使该能力生效（类似通过 `process.env.NODE_ENV` 来区分开发或生产环境）：

```ts
replace({
  'window.DISABLE_STYLE_INJECT': 'true',
}),
```

同时由于 `typeof true === 'undefined'` 恒为 `false`，此行代码及未用到的样式可在代码优化过程中被直接删除。

## 总结

相比于已经存在了数年，插件无比丰富的 webpack 来说，vite 并非完美无缺。但其优秀的性能，简约的实现与易于调试的插件，能让人更有欲望去探究并改造代码的底层逻辑。而踩坑与填坑的过程，也是一个深入理解代码的过程。

## 参考

+ https://github.com/vitejs/vite/issues/1579
+ https://cn.vitejs.dev/guide/api-plugin.html#plugin-ordering
