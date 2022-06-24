# 微信小程序 WebAssembly 开发指南

## 创建 `rustwasm` 项目

参考 [wasm-pack book](https://rustwasm.github.io/docs/wasm-pack/quickstart.html)，创建一个 `rustwasm` 项目。

## 编写 `rust` 代码

这里以一个简单的加法函数为例。

```rust
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    utils::set_panic_hook();
    a + b
}
```

`utils::set_panic_hook();`用于打印 `panic` ，参考 [wasm book](https://rustwasm.github.io/docs/book/reference/debugging.html#logging-panics) 。

## 构建 `wasm`

设置编译参数。

```toml
[profile.release]
# Tell `rustc` to optimize for small code size.
opt-level = "s"
lto = true
```

`lto = true` 用于缩小 `wasm` 包体积。

构建 `wasm` 包。
```shell
wasm-pack build --target web
```

在 `pkg` 目录下可见以下文件：

```shell
total 120
drwxr-xr-x   9 mryao  staff   288B Jun 24 17:38 .
drwxr-xr-x  17 mryao  staff   544B Jun 24 17:38 ..
-rw-r--r--   1 mryao  staff     1B Jun 24 17:38 .gitignore
-rw-r--r--   1 mryao  staff   2.9K Jun 24 17:38 README.md
-rw-r--r--   1 mryao  staff   263B Jun 24 17:38 package.json
-rw-r--r--   1 mryao  staff   1.1K Jun 24 17:38 wasm_wxa.d.ts
-rw-r--r--   1 mryao  staff   5.7K Jun 24 17:38 wasm_wxa.js
-rw-r--r--   1 mryao  staff    32K Jun 24 17:38 wasm_wxa_bg.wasm
-rw-r--r--   1 mryao  staff   326B Jun 24 17:38 wasm_wxa_bg.wasm.d.ts
```

进一步压缩 `wasm` 包体积：

```shell
brotli pkg/wasm_wxa_bg.wasm
```

得到 `wasm_wxa_bg.wasm.br` 。 `brotli` 压缩相关内容请参考 [GitHub brotli](https://github.com/google/brotli) 。

## 适配微信小程序环境

由于微信小程序环境的特殊性，需要对构建产物进行一些调整。调整主要集中在 `wasm_wxa.js` 上。

### 将 `WebAssembly` 替换为 `WXWebAssembly`

把 `wasm_wxa.js` 中的 `WebAssembly` 全部替换为 `WXWebAssembly` 即可。

### 删去不必要的 `import.meta`

微信小程序只允许调用本地 `wasm` 包：

```text
WXWebAssembly.instantiate(path, imports) 方法，path为代码包内路径（支持 .wasm 和.wasm.br后缀）
```

需要删去网络请求的代码：

```javascript
async function init(input) {
    // if (typeof input === 'undefined') {
    //     input = new URL('wasm_wxa_bg.wasm', import.meta.url);
    // }
    const imports = getImports();

    // if (typeof input === 'string' || (typeof Request === 'function' && input instanceof Request) || (typeof URL === 'function' && input instanceof URL)) {
    //     input = fetch(input);
    // }

    initMemory(imports);

    const { instance, module } = await load(await input, imports);

    return finalizeInit(instance, module);
}
```

```javascript
async function load(module, imports) {
    // if (typeof Response === 'function' && module instanceof Response) {
    //     if (typeof WXWebAssembly.instantiateStreaming === 'function') {
    //         try {
    //             return await WXWebAssembly.instantiateStreaming(module, imports);

    //         } catch (e) {
    //             if (module.headers.get('Content-Type') != 'application/wasm') {
    //                 console.warn("`WXWebAssembly.instantiateStreaming` failed because your server does not serve wasm with `application/wasm` MIME type. Falling back to `WXWebAssembly.instantiate` which is slower. Original error:\n", e);

    //             } else {
    //                 throw e;
    //             }
    //         }
    //     }

    //     const bytes = await module.arrayBuffer();
    //     return await WXWebAssembly.instantiate(bytes, imports);

    // } else {
    const instance = await WXWebAssembly.instantiate(module, imports);

    // if (instance instanceof WXWebAssembly.Instance) {
    //     console.log('hello');
    //     return { instance, module };

    // } else {
    return instance;
    // }
    // }
}
```

### 使用 `polyfill` 替换 `TextEncoder` 和 `TextDecoder`

`wasm_wxa.js` 中使用了 `TextEncoder` 和 `TextDecoder` ，微信小程序环境没有实现这两个类，需要用 `polyfill` 替换。 `polyfill` 相关内容参考 [GitHub MDN Commit History](https://github.com/mdn/content/commit/53f54d155b82343ae43ade6097f4b2b9d651036f) 。

引入 [EncoderDecoderTogether](https://github.com/anonyco/FastestSmallestTextEncoderDecoder/blob/master/EncoderDecoderTogether.min.js)：

```javascript
require('./EncoderDecoderTogether.min');
const { TextEncoder, TextDecoder } = global;
```

## 在微信小程序中使用 wasm

<div align="center">

  <h1><code>wasm-pack-template</code></h1>

<strong>A template for kick starting a Rust and WebAssembly project
using <a href="https://github.com/rustwasm/wasm-pack">wasm-pack</a>.</strong>

  <p>
    <a href="https://travis-ci.org/rustwasm/wasm-pack-template"><img src="https://img.shields.io/travis/rustwasm/wasm-pack-template.svg?style=flat-square" alt="Build Status" /></a>
  </p>

  <h3>
    <a href="https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/index.html">Tutorial</a>
    <span> | </span>
    <a href="https://discordapp.com/channels/442252698964721669/443151097398296587">Chat</a>
  </h3>

<sub>Built with 🦀🕸 by <a href="https://rustwasm.github.io/">The Rust and WebAssembly Working
Group</a></sub>
</div>

## About

[**📚 Read this template tutorial! 📚**][template-docs]

This template is designed for compiling Rust libraries into WebAssembly and
publishing the resulting package to NPM.

Be sure to check out [other `wasm-pack` tutorials online][tutorials] for other
templates and usages of `wasm-pack`.

[tutorials]: https://rustwasm.github.io/docs/wasm-pack/tutorials/index.html

[template-docs]: https://rustwasm.github.io/docs/wasm-pack/tutorials/npm-browser-packages/index.html

## 🚴 Usage

### 🐑 Use `cargo generate` to Clone this Template

[Learn more about `cargo generate` here.](https://github.com/ashleygwilliams/cargo-generate)

```
cargo generate --git https://github.com/rustwasm/wasm-pack-template.git --name my-project
cd my-project
```

### 🛠️ Build with `wasm-pack build`

```
wasm-pack build
```

### 🔬 Test in Headless Browsers with `wasm-pack test`

```
wasm-pack test --headless --firefox
```

### 🎁 Publish to NPM with `wasm-pack publish`

```
wasm-pack publish
```

## 🔋 Batteries Included

* [`wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen) for communicating
  between WebAssembly and JavaScript.
* [`console_error_panic_hook`](https://github.com/rustwasm/console_error_panic_hook)
  for logging panic messages to the developer console.
* [`wee_alloc`](https://github.com/rustwasm/wee_alloc), an allocator optimized
  for small code size.
