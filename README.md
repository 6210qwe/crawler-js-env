# crawler-js-env

爬虫 JS 逆向的「补环境」工作区。把目标站点混淆过的加密逻辑（`sign`、`_signature`、`a_bogus` 之类）搬进 Node 里跑通，
从而在不启动浏览器的前提下复现请求参数。

内核是 [HaHaVM](HaHaVM-main/)（零点开发，MIT License，本仓库整包引入并做本地改动），
外层是一个可直接运行的入口和配置约定。

## 为什么需要补环境

目标站点的加密函数通常依赖 DOM / BOM：

- `document.createElement('canvas').toDataURL()` —— 指纹
- `window.navigator.userAgent` / `screen` / `plugins` —— 环境指纹
- `location.href`、`localStorage`、`XMLHttpRequest` —— 上下文与网络层
- `setTimeout` / `Promise` —— 异步分支

直接 `node xxx.js` 会因为这些对象 undefined 而报错，或者走不到加密分支。
补环境就是**照原样伪造这些对象**，让代码认为自己在浏览器里。

## 目录

```
crawler-js-env/
├── package.json          依赖（vm2）
├── HaHaVM-main/          补环境内核
│   ├── main.js           入口：拼装沙箱并执行
│   ├── config/           config / env / tools / user 四组配置
│   ├── env/              浏览器对象实现（Document、Canvas、Navigator…）
│   ├── tools/            工具函数
│   ├── user/             每个目标站点一个目录（自己创建）
│   └── 笔记.md           开发过程笔记
```

## 快速开始

```bash
npm install
cd HaHaVM-main
# 在 user/ 下建目标目录，按执行顺序放好这些文件：
#   globalVar.js -> userVar.js -> domVar.js -> proxyObj.js -> input.js -> async.js -> myExec.js
# 然后在 main.js 里把 const name = "test" 改成你的目录名
node main.js
```

每个文件的职责：

| 文件 | 作用 |
|---|---|
| `globalVar.js` | 全局变量初始化（`window`、`self`、`top`） |
| `userVar.js` | 目标站点自己挂到 window 上的变量 |
| `domVar.js` | DOM 结构伪造（元素、属性、样式） |
| `proxyObj.js` | `Proxy` 拦截，补漏掉的属性访问 |
| `input.js` | 目标加密代码本体 |
| `async.js` | 异步分支需要的实现 |
| `myExec.js` | 出参：打印 / 返回加密结果 |

`output.js` 是所有片段拼接后的整体，可以直接用 node 执行验证。

## 补环境的通用套路

1. **先跑，看报错**。`ReferenceError: xxx is not defined` 就补 `xxx`。
2. **打 Proxy 兜底**。给 `window`、`document` 套 Proxy，`get` 时把未定义的 key 记进日志，比一条条试快得多。
3. **返回值要像真的**。`navigator.plugins.length` 返回 0 和返回 5，指纹完全不一样；该给的数组结构要给全。
4. **异步要带时间**。`setTimeout` 直接同步执行会导致顺序错乱，用 `queueMicrotask` / 真定时器。
5. **验证一致性**。同一个输入跑十次结果必须稳定，否则说明有随机源（时间、Math.random）没固定。

## 已知问题

- `vm2` 已在 2023 年停止维护，且存在沙箱逃逸漏洞（CVE-2023-37466）。本仓库只跑自己可控的 JS 代码，风险可接受；
  如果要跑不可信代码，换 `isolated-vm`。
- `vm2` 的 `WebAssembly` 支持有限，遇到 wasm 加密需要换 `node:vm` + 自建 `WebAssembly` 补丁。

## 出处与授权

本项目的补环境内核来自**零点大佬的补环境框架（HaHaVM）**，我在其基础上做的完善：

- 拆掉仓库里误提交的 `node_modules/` 与 `.idea/`，改为 `npm install` 复现依赖
- 重写 `package.json`，固定入口脚本与 Node 版本要求
- 补充 `user/` 目录的执行顺序约定与各文件职责说明（原仓库只有一句顺序提示）
- 整理补环境的通用套路与已知风险（vm2 沙箱逃逸、wasm 支持缺失）
- 把「先跑看报错 → Proxy 兜底 → 指纹一致性校验」这套排查流程写成文档

原始框架：[HaHaVM](https://github.com/yitiaohuangxianyu/HaHaVM)，作者零点，MIT License，`HaHaVM-main/LICENSE` 原样保留。

本项目仅用于逆向技术学习与自有业务的数据采集，请遵守目标站点服务条款与相关法律。
