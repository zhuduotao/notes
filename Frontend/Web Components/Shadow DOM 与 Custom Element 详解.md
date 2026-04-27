---
created: '2026-04-27'
updated: '2026-04-27'
tags:
  - web-components
  - shadow-dom
  - custom-elements
  - frontend
  - browser-api
aliases:
  - Shadow DOM
  - Custom Elements
  - 自定义元素
  - 影子 DOM
source_type: official-doc
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/API/Web_components'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM
  - >-
    https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements
  - 'https://dom.spec.whatwg.org/#shadow-trees'
  - 'https://html.spec.whatwg.org/multipage/custom-elements.html'
status: verified
---

Web Components 是一套浏览器原生技术的组合，用于创建可封装、可复用的自定义 HTML 元素。它由三项核心技术构成：**Custom Elements（自定义元素）**、**Shadow DOM（影子 DOM）** 和 **HTML Templates（HTML 模板）**。本文重点介绍前两项。

## 是什么

### Custom Element（自定义元素）

允许开发者通过 JavaScript 类定义全新的 HTML 元素（或扩展已有元素），并注册到浏览器中，使其像原生标签一样在 HTML 中使用。

### Shadow DOM（影子 DOM）

允许将一个隐藏的 DOM 子树附加到任意元素上，该子树与主文档 DOM 隔离——外部 CSS 和 JavaScript 无法直接影响其内部，从而实现真正的样式和行为封装。

两者配合使用，构成了 Web Components 的基石：Custom Element 定义"这个元素是什么、有什么行为"，Shadow DOM 保证"这个元素的内部实现不会被外界意外破坏"。

## 为什么重要

- **原生封装**：不依赖任何框架或构建工具，浏览器原生支持样式和作用域隔离
- **跨框架复用**：一个 Custom Element 可以在 React、Vue、Angular 或纯 HTML 中直接使用
- **避免样式冲突**：Shadow DOM 的 CSS 作用域隔离解决了大型项目中常见的样式污染问题
- **微前端场景**：多个独立应用嵌入同一页面时，Shadow DOM 是隔离各应用 DOM 和 CSS 的关键手段

## Custom Element 详解

### 两种类型

| 类型 | 继承自 | 使用方式 | 浏览器支持 |
|------|--------|----------|------------|
| **Autonomous（自主元素）** | `HTMLElement` | 直接使用自定义标签名，如 `<my-element>` | 全浏览器支持 |
| **Customized Built-in（定制内置元素）** | 具体元素接口如 `HTMLParagraphElement` | 通过 `is` 属性扩展已有元素，如 `<p is="my-paragraph">` | Safari 不支持，不推荐用于生产环境 |

### 定义与注册

```js
// 1. 定义类（继承 HTMLElement）
class PopupInfo extends HTMLElement {
  constructor() {
    super(); // 必须首先调用
    // 可设置初始状态、默认值、事件监听
    // 注意：构造函数中不应访问属性或子元素
  }

  connectedCallback() {
    // 元素被插入文档时调用
    // 推荐：大部分初始化逻辑放在此处而非 constructor
  }

  disconnectedCallback() {
    // 元素从文档移除时调用
    // 用于清理：移除事件监听、取消定时器等
  }

  adoptedCallback() {
    // 元素被移动到新的 document 时调用
  }

  attributeChangedCallback(name, oldValue, newValue) {
    // 属性变化时调用
    // 仅在 observedAttributes 中声明的属性才会触发
  }
}

// 2. 注册到全局 CustomElementRegistry
customElements.define("popup-info", PopupInfo);
```

### 生命周期回调

| 回调 | 触发时机 | 典型用途 |
|------|----------|----------|
| `constructor()` | 元素实例创建时 | 初始化 Shadow DOM、绑定事件监听器 |
| `connectedCallback()` | 元素被插入文档时 | 渲染内容、获取属性值、发起请求 |
| `disconnectedCallback()` | 元素从文档移除时 | 清理资源、移除全局事件监听 |
| `adoptedCallback()` | 元素跨 document 移动时 | 处理跨窗口/iframe 场景 |
| `attributeChangedCallback()` | 监听属性变化时 | 根据属性更新内部状态或 UI |
| `connectedMoveCallback()` | 使用 `Element.moveBefore()` 移动元素时 | 替代 connected/disconnected 以避免重复初始化/清理 |

### 属性监听

通过 `static observedAttributes` 声明需要监听的属性：

```js
class MyElement extends HTMLElement {
  static observedAttributes = ["color", "size"];

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`${name}: ${oldValue} -> ${newValue}`);
  }
}
```

**注意**：如果 HTML 中已声明了被监听的属性，`attributeChangedCallback` 会在 DOM 解析完成后立即触发一次。

### 自定义状态（Custom States）

自主元素可通过 `ElementInternals` 定义内部状态，并用 CSS `:state()` 伪类选择器匹配：

```js
class MyElement extends HTMLElement {
  constructor() {
    super();
    this._internals = this.attachInternals();
  }

  get collapsed() {
    return this._internals.states.has("hidden");
  }

  set collapsed(flag) {
    flag
      ? this._internals.states.add("hidden")
      : this._internals.states.delete("hidden");
  }
}

customElements.define("my-element", MyElement);
```

```css
my-element { border: dashed red; }
my-element:state(hidden) { border: none; }
```

### 作用域注册表（Scoped Custom Element Registries）

解决多组件库之间的命名冲突问题。每个 Shadow Root 可以使用独立的注册表：

```js
// 创建独立注册表
const myRegistry = new CustomElementRegistry();

myRegistry.define(
  "my-element",
  class extends HTMLElement {
    connectedCallback() {
      this.textContent = "Hello from scoped registry!";
    }
  }
);

// 附加到 Shadow Root
const host = document.createElement("div");
const shadow = host.attachShadow({
  mode: "open",
  customElementRegistry: myRegistry,
});

shadow.innerHTML = "<my-element></my-element>";
```

**限制**：作用域注册表不支持 `extends` 选项（即不支持定制内置元素）。

## Shadow DOM 详解

### 核心术语

| 术语 | 说明 |
|------|------|
| **Shadow Host** | 附加 Shadow DOM 的普通 DOM 节点 |
| **Shadow Root** | Shadow DOM 树的根节点 |
| **Shadow Tree** | Shadow DOM 内部的 DOM 树 |
| **Shadow Boundary** | Shadow DOM 与普通 DOM 的分界线 |

### 创建方式

#### 命令式（JavaScript）

```js
const host = document.querySelector("#host");
const shadow = host.attachShadow({ mode: "open" });

const span = document.createElement("span");
span.textContent = "I'm in the shadow DOM";
shadow.appendChild(span);
```

#### 声明式（HTML）

```html
<div id="host">
  <template shadowrootmode="open">
    <span>I'm in the shadow DOM</span>
  </template>
</div>
```

浏览器解析时会自动将 `<template>` 替换为附加到父元素的 Shadow Root。

### mode 选项

| mode | 行为 | 适用场景 |
|------|------|----------|
| `"open"` | 可通过 `element.shadowRoot` 访问 Shadow DOM | 大多数场景，便于调试和外部交互 |
| `"closed"` | `element.shadowRoot` 返回 `null` | 强封装场景，但非真正的安全机制（浏览器扩展等仍可访问） |

### CSS 封装

Shadow DOM 提供双向 CSS 隔离：

- **外部样式不影响内部**：页面中的 `span { color: blue; }` 不会影响 Shadow DOM 内的 `<span>`
- **内部样式不影响外部**：Shadow DOM 内的样式不会泄漏到页面其他部分

#### Shadow DOM 内部样式方案

**方案一：Constructable Stylesheets（推荐，可共享）**

```js
const sheet = new CSSStyleSheet();
sheet.replaceSync("span { color: red; }");

const shadow = host.attachShadow({ mode: "open" });
shadow.adoptedStyleSheets = [sheet];
```

优势：同一样式表可在多个 Shadow Root 间共享，浏览器只解析一次。

**方案二：`<style>` 元素（简单场景）**

```js
const style = document.createElement("style");
style.textContent = "span { color: red; }";
shadow.appendChild(style);
```

#### Shadow DOM 中的 CSS 选择器

| 选择器 | 作用 |
|--------|------|
| `:host` | 选择 Shadow Host 本身 |
| `:host(.active)` | 仅当 Host 匹配 `.active` 时选择 |
| `:host-context(.theme-dark)` | 当 Host 的祖先匹配时选择 |
| `::slotted(*)` | 选择被插入到 `<slot>` 中的内容 |
| `::part(name)` | 选择带有 `part="name"` 属性的 Shadow DOM 内部元素 |

### Slot（插槽）

`<slot>` 元素允许外部内容插入到 Shadow DOM 中：

```js
class MyElement extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: "open" });
    shadow.innerHTML = `
      <h2>标题</h2>
      <slot name="content">默认内容</slot>
    `;
  }
}
```

```html
<my-element>
  <span slot="content">自定义内容</span>
</my-element>
```

### 事件穿透

默认情况下，Shadow DOM 内部触发的事件不会冒泡到 Shadow Boundary 之外。若需要事件穿透：

```js
event = new CustomEvent("my-event", {
  bubbles: true,
  composed: true, // 允许事件穿越 Shadow Boundary
});
```

通过 `event.composedPath()` 可获取事件路径（不包含 `mode: "closed"` 的 Shadow DOM 内部节点）。

## Custom Element + Shadow DOM 组合示例

```js
class FilledCircle extends HTMLElement {
  constructor() {
    super();
  }

  connectedCallback() {
    const shadow = this.attachShadow({ mode: "open" });

    const svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    const circle = document.createElementNS(
      "http://www.w3.org/2000/svg",
      "circle"
    );
    circle.setAttribute("cx", "50");
    circle.setAttribute("cy", "50");
    circle.setAttribute("r", "50");
    circle.setAttribute("fill", this.getAttribute("color") || "black");
    svg.appendChild(circle);

    shadow.appendChild(svg);
  }
}

customElements.define("filled-circle", FilledCircle);
```

使用：

```html
<filled-circle color="blue"></filled-circle>
```

## 限制与注意事项

### Custom Element 构造函数约束

- 必须首先调用 `super()`
- 不应在构造函数中访问元素的属性或子元素（此时尚未解析）
- 不应在构造函数中添加属性或子元素
- 完整约束见 [WHATWG 规范](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-element-conformance)

### 命名规则

自定义元素名称必须：
- 以小写字母开头
- 包含至少一个连字符 `-`
- 不能是 HTML 规范中保留的名称（如 `font-face`）
- 详见 [有效自定义元素名称规范](https://html.spec.whatwg.org/multipage/custom-elements.html#valid-custom-element-name)

### Safari 不支持 Customized Built-in Elements

`<p is="my-paragraph">` 这种扩展内置元素的方式在 Safari 中不被支持，且 Safari 团队明确表示无计划支持。生产环境中建议使用自主元素。

### `<link>` 外部样式表的 FOUC 问题

在 Shadow DOM 中使用 `<link>` 引入外部样式表时，不会阻塞 Shadow Root 的渲染，可能出现无样式内容闪烁（FOUC）。现代浏览器对相同文本的 `<style>` 标签有优化（共享底层样式表），性能与外部样式表相当。

### `mode: "closed"` 不是安全机制

`mode: "closed"` 只是表明"页面不应访问内部"，但浏览器扩展等仍可绕过。不应将其视为安全边界。

### SSR 与 Declarative Shadow DOM

声明式 Shadow DOM（`<template shadowrootmode="open">`）支持服务端渲染，但属于较新的 API，需确认目标浏览器兼容性。

## 相关概念

- **HTML Templates（`<template>` + `<slot>`）**：Web Components 的第三项技术，用于定义可复用的 HTML 模板结构
- **Constructable Stylesheets**：可通过 `CSSStyleSheet` 构造函数创建并在多个 Shadow Root 间共享的样式表
- **ElementInternals**：提供对表单关联、无障碍访问（ARIA）和自定义状态的控制能力
- **微前端隔离**：qiankun、MicroApp 等微前端框架利用 Shadow DOM 实现 CSS 和 DOM 隔离

## 参考资料

- [MDN - Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components)
- [MDN - Using custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)
- [MDN - Using shadow DOM](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [MDN - Using templates and slots](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_templates_and_slots)
- [WHATWG DOM Standard - Shadow Trees](https://dom.spec.whatwg.org/#shadow-trees)
- [WHATWG HTML Standard - Custom Elements](https://html.spec.whatwg.org/multipage/custom-elements.html)
- [MDN Web Components Examples (GitHub)](https://github.com/mdn/web-components-examples)
