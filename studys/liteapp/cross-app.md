# 原生时代
移动端原生技术需要配置 IOS 和 Android 两套技术团队和技术栈
存在发版周期限制, 开发效率低, 开发成本高

# Hybrid 时代
"不管啥平台, 只要给一个浏览器, Js 就能跑"

在 IOS or Android 系统上运行 Js 不是难事, 但是还需要做到 H5
和原生平台的交互, 于是 JsBridge 诞生了

在原生平台中, 我们的 Js 代码运行在一个单独 Js Context 中 (比如 Webkit 引擎, JavascriptCore 等), 这个独立 Context 的和原生能力的交互是双向的
  1. Js 调用 Native
     1. 注入 APIs
     原生平台通过 WebView 提供的接口, 向 Js Context 中 (window 对象) 注入可操作的对象
     2. 拦截 URL Scheme
     前端直接发送定义好的 URL Scheme 请求, 原生平台拦截后, 做出反应
  2. Native 调用 Js
     原生平台可以通过 WebView 提供的接口 直接执行 Js 代码

Hybrid 框架: Cordova, Ionic
Hrbrid 面对的问题:
  * Js 上下文如果和 Native 异步通信频繁, 导致性能较差
  * 页面渲染是前端做的, 组件也是, 组件的性能不如原生组件
  * WebView 内核在各平台上不统一, 国内厂商喜欢深度定制

# RN 时代
用 Web 语言做开发, 用原生平台做渲染, 代表: React Native, Weex 等
这样一来, 就摆脱了 WebView 内核的束缚, 又保障了开发体验和效率和渲染性能

本质是将虚拟 DOM 的渲染 API 由 Web DOM APIs 替换为原生平台 APIs
通过这项技术就可以用同一套框架 (比如 React), 同时开发出原生和 H5 应用

前端和原生的衔接是通过 JavaScriptCore 这样的 JS 内核库实现双向通信
JavaScriptCore 解析 Js 代码, 封装信息传递给原生平台
  比如 VDOM Patch 过后的更新视图, 比如触发事件, 网络请求
原生平台做好渲染, 请求, 事件等等逻辑, 通过 JavaScriptCore 直接执行 Js

其实微信小程序就也是这种东西, 只不过依托微信平台提供了更为完备的原生能力支持

现实很骨感:
  1. 前端代码与原生平台异步的通信成本变得非常高
  2. Js 逻辑与原生能力是线程隔离的, 无法共享一个内存空间
  3. 平台 API 差异是无法用一套框架完全抹平的 (小程序可以)

# Flutter 时代
全新的编程语言, 全新的渲染引擎
在 RN 里, 一个 <view> 组件会被渲染成原生的 UIView Element (IOS) 或者 View Element (Android)
但 Flutter 自己提供了一套组件集合, 这些组件的渲染被 Flutter 渲染引擎直接接管

Flutter 组件分为两种类型:
  StatelessWidget 无状态组件
  StatefulWidget  有状态组件

Flutter 的渲染:
Skia: 一个 2D 绘图引擎库, 能跨平台
Skia 唯一需要的就是原生平台提供 Canvas 接口, 来做绘制

貌似后来小程序的原生渲染, 也改成了利用 Flutter 的方式, 而不再使用原生平台组件

目前来看, Flutter 优势最大, 前景一片大好
