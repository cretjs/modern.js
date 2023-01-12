# Esbuild 插件

:::tip esbuild 介绍
[esbuild](https://esbuild.github.io/) 是一款基于 Golang 开发的前端构建工具，具有打包、编译和压缩 JavaScript 代码的功能，相比传统的打包编译工具，esbuild 在性能上有显著提升。在代码压缩方面，相比 webpack 内置的 terser 压缩器，esbuild 在性能上有数十倍的提升。
:::

Builder 提供了 esbuild 插件，让你能使用 esbuild 代替 babel-loader、ts-loader 和 terser 等库进行代码编译和压缩。在大型工程中启用 esbuild 后，**可以大幅度减少代码编译和压缩所需的时间，同时有效避免 OOM (heap out of memory) 问题**。

## 快速开始

### 安装插件

你可以通过如下的命令安装插件:

```bash
# npm
npm install @modern-js/builder-plugin-esbuild -D

# yarn
yarn add @modern-js/builder-plugin-esbuild -D

# pnpm
pnpm add @modern-js/builder-plugin-esbuild -D
```

### 注册插件

在 Modern.js / EdenX 等上层框架中，你可以通过 `builderPlugins` 配置项来注册 esbuild 插件：

```ts
import { PluginEsbuild } from '@modern-js/builder-plugin-esbuild';

export default {
  builderPlugins: [PluginEsbuild()],
};
```

当你直接调用 Builder 的 Node API 时，可以通过 `addPlugins` 方法来注册 esbuild 插件：

```js
import { PluginEsbuild } from '@modern-js/builder-plugin-esbuild';

// 往 builder 实例上添加插件
builder.addPlugins([PluginEsbuild()]);
```

## 配置

插件默认会开启代码转译和代码压缩的功能，你也可以通过配置来自定义插件的行为。

### loader

- **Type**:

```ts
type LoaderOptions = EsbuildLoaderOptions | false | undefined;
```

- **Default**:

```ts
const defaultOptions = {
  target: 'es2015',
  charset: builderConfig.output.charset,
};
```

这个选项用于启用 JavaScript 和 TypeScript 的转译，启用时将会使用 esbuild-loader 替换 babel-loader 和 ts-loader。

如果你需要修改转译参数，可以查看 [esbuild-loader 文档](https://github.com/privatenumber/esbuild-loader#loader)。

#### 修改目标环境

通过 `target` 选项来修改代码转译的目标环境。`target` 可以直接设置为 JavaScript 语言版本，比如 `es6`，`es2020`；也可以设置为若干个目标环境，每个目标环境都是一个环境名称后跟一个版本号，比如 `['chrome58', 'edge16' ,'firefox57']`。`target` 字段的详细介绍可以参考 [esbuild - target](https://esbuild.github.io/api/#target)。

target 支持设置为以下环境：

- chrome
- edge
- firefox
- ie
- ios
- node
- opera
- safari

```ts
PluginEsbuild({
  loader: {
    target: 'chrome61',
  },
});
```

#### 关闭代码转译

将 `loader` 设置为 `false` 来关闭 esbuild 代码转译，此时 Builder 会继续使用 Babel 来进行代码转译。

```ts
PluginEsbuild({
  loader: false,
});
```

### minimize

- **Type**:

```ts
type MinimizeOptions = EsbuildMinifyOptions | false | undefined;
```

- **Default**:

```ts
const defaultOptions = {
  css: true,
  target: 'es2015',
};
```

这个选项用于启用 JavaScript 和 CSS 的代码压缩。

如果你需要修改压缩参数，可以查看 [esbuild-loader 文档](https://github.com/privatenumber/esbuild-loader#minifyplugin)。

#### 修改目标环境

通过 `target` 选项来修改代码压缩的目标环境。

```ts
PluginEsbuild({
  minimize: {
    target: 'chrome61',
  },
});
```

#### 关闭代码压缩

将 `minimize` 设置为 `false` 来关闭 esbuild 代码压缩，此时 Builder 会继续使用 Terser 进行代码压缩。

```ts
PluginEsbuild({
  minimize: false,
});
```

## esbuild 局限性

虽然 esbuild 能给现有的 webpack 项目带来明显的构建性能提升，但这个工具在接入 Builder 时还存在一定的局限性，需要大家在接入的时候格外注意。

### 兼容性

使用 esbuild 进行代码转译时（即 `loader` 能力），esbuild 通常最低支持到 ES2015（即 ES6）语法，并且不具备自动注入 Polyfill 的能力。如果生产环境需要降级到 ES5 及以下的语法，建议使用 SWC 编译。

你可以通过如下的配置指定目标语法版本:

```ts
PluginEsbuild({
  loader: {
    target: 'es2015',
  },
});
```

使用 esbuild 进行代码压缩时（即 `minimize` 能力），esbuild 可以在生产环境中进行压缩和混淆，通常最低支持到 ES2015 语法。

如果设置压缩的 `target` 为 `es5`，需要保证所有代码已经被转义为 ES5 代码，否则会导致 esbuild 编译报错：`Transforming 'xxx' to the configured target environment ("es5") is not supported yet`。

因此，对于生产环境需要兼容 ES5 及以下语法的项目，请谨慎开启 minimize 能力，建议使用 SWC 压缩。

你可以通过如下的配置指定目标语法版本:

```ts
PluginEsbuild({
  minimize: {
    target: 'es2015',
  },
});
```

### 不支持 Babel 插件

使用 esbuild 进行代码转译时，诸如 `babel-plugin-import` 等原有 Babel 插件的语法编译功能在开启 esbuild 后无法使用。并且由于 Builder 底层使用的是 esbuild 的 `Transform API`，因此不支持使用额外 esbuild 插件来进行自定义编译过程。

如果你有 `babel-plugin-import` 等 Babel 插件相关诉求，可以使用 SWC 插件。

### 产物体积

使用 esbuild 压缩虽然带来了构建效率上的提升，但 esbuild 的压缩比例是低于 terser 的，因此**构建产物的体积会增大**，请根据业务情况酌情使用。通常来说，esbuild 比较适合中后台等对体积不敏感的场景。

对于压缩工具之间的详细对比，可以参考 [minification-benchmarks](https://github.com/privatenumber/minification-benchmarks)。