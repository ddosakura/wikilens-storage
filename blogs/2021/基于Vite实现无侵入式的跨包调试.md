# 基于 Vite 实现无侵入式的跨包调试

> 在同时开发多个包时，跨包调试曾是一件麻烦事，本文借助 Vite 实现了一种无侵入式的跨包调试方案。

## 背景

以下是一个常见的单体式仓库目录结构：

```
# tree packages 
packages
├── example-a
│   ├── dist
│   │   ├── index.es.js
│   │   ├── index.umd.js
│   ├── src
│   │   ├── index.ts
│   ├── index.html
│   ├── package.json
├── example-b
│   ├── dist
│   │   ├── index.es.js
│   │   ├── index.umd.js
│   ├── src
│   │   ├── index.ts
│   ├── index.html
│   ├── package.json
└── shared
    ├── dist
    │   ├── index.es.js
    │   ├── index.umd.js
    ├── src
    │   ├── index.ts
    ├── index.html
    ├── package.json
```

而一般在 `package.json` 中，会指定 `dist` 目录下经过打包后的文件为入口文件：

```json
# package.json
{
  "main": "./dist/index.umd.js",
  "module": "./dist/index.es.js",
  "exports": {
    ".": {
      "import": "./dist/index.es.js",
      "require": "./dist/index.umd.js"
    }
  }
}
```

当依赖包 `shared` 的源码更改后，假如没有进行构建，`dist` 目录中的代码并不会变更；更关键的是，一个刚 `clone` 下来的仓库，由于没有 `dist` 目录，即使没进行任何代码修改，也无法运行一个有依赖的单一子包。

而不论是修改依赖包的入口，还是修改另一子包的 `import` 路径，都会对代码进行侵入式的修改，十分不方便；而在非单体式仓库的情况下，甚至需要改子包的 `package.json` 中的依赖为本地包的路径。

## vite 重定向源码插件的实现

vite 在开发环境下，可以访问到文件系统上有访问权限的任何一个文件。这也为我们重定向源码提供了方便。

比如，源码中一个简单的引入：

```ts
import vue from "vue";
```

翻译到浏览器中的 esm 模块后被补全为：

```js
import vue from "/node_modules/.vite/vue.js?v=<hash>";
```

或是补全为文件系统绝对路径：

```js
import vue from "/@fs/.../node_modules/.vite/vue.js?v=<hash>";
```

基于这一知识，我们可以通过插件直接劫持需要调试的包的入口指向我们的源码！

插件核心转换逻辑如下：

```ts
{
  transform(_src, id) {
    const isResolve = resolve.test(id);
    const isExclude = exclude.test(id);
    if (!isResolve || isExclude) return;
    // 将打包后的入口文件的代码替换为从源码原样导出
    const code = `export * from '/@fs${id.replace(resolve, replace)}'`;
    return code;
  },
}
```

同时，需要注意，本转换仅应该应用于 `serve` 命令下（开发环境）。

## 插件的应用

### 单体式仓库

一个最简单的应用是用于单体式仓库。

```ts
const rps = redirectPackageSrc({
  resolve: /(\/packages\/.*\/)dist\/.*$/g,
  replace: ($0, $1) => `${$1}src/index.ts`,
});
```

我们将所有匹配单体式仓库子包入口的文件重定向到子包的源码入口，然后就可以在不预先构建子包的状态下开发拉！

### 多仓库

对于多仓库，常见的调试方法需要更改 `package.json` 的依赖为本地包：

```json
{
  "dependencies": {
    "a-example-lib": "file:<path>"
  },
}
```

但是，通过重定向源码的方式，我们只需要在插件中写明地址，不再需要倾入式的更改 `package.json` 文件：

```ts
const rps = redirectPackageSrc({
  resolve: /(\/a-example-lib\/.*\/)dist\/.*$/g,
  // 替换为该依赖在文件系统中的绝对路径
  replace: ($0, $1) => `/root/xxx${$1}src/index.ts`,
});
```

需要注意，出于安全考虑，vite 不一定能访问到该路径下的文件，需要在配置中声明对该路径的授权：

```ts
export default defineConfig({
  server: {
    fs: {
      allow: [
        '.',
        '/root/xxx',
      ],
    },
  },
};
```

详见官方文档：https://cn.vitejs.dev/config/index.html#server-fs-allow

## 后记（与 alias 的对比）

针对简单的外部仓库别名替换，使用 alias 也可以满足需求:

```ts
{
  find: /(a-example-lib)/, // 注意这里匹配的是经过 vite 转换前的包名，可使用正则
  replacement: '<pathoflib>/$1/lib/index.ts',
}
```

+ https://vitejs.dev/config/#resolve-alias
+ https://github.com/rollup/plugins/tree/master/packages/alias#entries

本文方案与 alias 的区别在于，alias 匹配的是经过 vite 转换前的包名然后进行替换，而本文方案替换的是匹配的入口文件的内容。

在单体式仓库的情况下，使用 alias 会面临许多复杂的情况：
+ 包名不一定方便匹配（包名和文件名不一定一致、存在不同的命名空间、同一命名空间下可以还有 repo 外的项目）
+ 即使包名和文件名一致，提前扫描文件目录生成的 alias 也不方面动态创建子包（即需要重启服务）

本文方案的优势在于匹配的是入口文件的 id，该 id 无论是通过包名、相对路径，亦或者在 vite 转换后的路径为非绝对路径的 `/packages/shared/dist/index.es.js` 等各种情况下，其在导入后 transform 阶段的 id 是一致且易于匹配的（`/@fs/xxx/packages/yyy/dist/index.es.js`）。同时其与该包名无关，且可以通过相对路径得到其源码位置（一般为 `/@fs/xxx/packages/yyy/src/index.ts`）
