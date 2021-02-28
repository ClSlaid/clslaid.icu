# 俺如何为 hugo-theme-even 提供 mermaid 支持


{{<mermaid>}}
sequenceDiagram
participant 主机A
participant 主机B
主机A ->> 主机B: 发送 SYN (seq = x)
Note right of 主机B: 收到 SYN (seq = x)
主机B -->> 主机A: 发送 SYN (seq = y)<br/>发送 ACK (ack = x+1)
Note Left of 主机A: 收到 SYN (seq = y)<br/>收到 ACK (ack = x+1)
主机A ->> 主机B: 发送 ACK (ack = y+1)
Note right of 主机B: 收到 ACK (ack = y+1)
Note over 主机A, 主机B: TCP 连接建立成功
{{</mermaid >}}


如上是一个利用 `mermaid` 绘制的 SVG 图形 ，相比于直接在某些软件绘制好再截图，显然利用 `mermaid` 绘图具备诸多优点：

1. 网站保存的是带 `mermaid` 的 `markdown` 文件，不涉及到图片存储。存储占用小。
2. 渲染出图是 `SVG` 格式，上面的文字可以直接拷贝。
3. 网站数据传输时是使用 `mermaid` 源码，而不是图片，可以有效减少网站加载过程产生的流量耗费。

## 如何为你的 `Hugo` 博客引入 `mermaid` 支持？

{{<admonition tip "贴士" true>}}
虽然由于 even theme 更新过慢，似乎久已无更多维护，因此本站已经开始使用 LoveIt 作为主题。

但是对于 even theme 而言，本方法应当依旧适用。
{{</admonition>}}

对于这一点，俺可以说前人之述备矣。反手就是一堆参考资料：

- [Diagrams in hugo with mermaid](https://skeptric.com/diagrams-in-hugo/)
- [拓展 Hugo 的 MarkDown 流程图](https://kentxxq.com/contents/%E6%8B%93%E5%B1%95hugo%E7%9A%84markdown_%E6%B5%81%E7%A8%8B%E5%9B%BEmermaid/)

这些解决方案引入 `mermaid` 的方法简单素雅，这一套流程可以总结为：

1. 在博客中加载 `mermaidJS`。具体方法有：

    - 直接在一定会加载的 `HTML` 文件中引入 `<script>` 标签，加载 `mermaid`。
    - 在部分支持自定义 `JavaScript` 的主题中，引入 `JavaScript` 文件。

2. 自定义 `ShortCodes`，依靠 `ShortCodes` 实现标签替换。

但是习惯了通过 `typora` 之类的 `MarkDown` 拓展编辑器，他们无一例外都支持按照如下风格进行渲染：

    ```mermaid

    ... 图表内容 ...

    ```

能不能实现它，从而实现更加一颗赛艇的编辑体验呢？

**能的**。

---

## 实现原理

通过阅读 `mermaid` 的 API 咱们可以得知 `mermaidAPI` 渲染 `mermaid` 图形的大致流程：

{{<mermaid>}}
sequenceDiagram
participant mermaid
Note over 网页内容: MarkDown 文件
其他支持 -->> 网页内容: 渲染 MarkDown
Note over 网页内容: HTML 文件
Note over 网页内容: 此时 mermaid 图表代码<br/>被包含在特定标签中
Note over mermaid: mermaidAPI.initialize()
mermaid -> 网页内容: 查找、获取特定标签内容
loop 渲染 mermaid 图表
mermaid -->> mermaid: 将特定标签中的内容进行渲染
end
mermaid ->> 网页内容: 替换相应内容
Note over 网页内容: 内容被替换为 SVG
其他支持 -->> 网页内容: 其他内容渲染
Note over 网页内容: 渲染完成的 HTML 文件
{{</mermaid>}}

上述的**特定标签**是指\<div class=\"mermaid">的 `HTML` 标签。通过检查页面元素，咱们会发现如今的大部分博客将代码块渲染成如下式样：

```HTML
<pre><code class="language-语言种类">
    <!-- code -->
</code></pre>
```

也就是说咱们只需要找到形如：

```HTML
<pre><code class="language-mermaid">
    <!-- mermaid code -->
</code></pre>
```

的代码块，并将其替换成``<div class="mermaid">``即可。

---

## 在 even theme 中的实现

{{<admonition warning "注意">}}
注意： 下文中的 “/” 目录代表 even theme 项目的根目录而不是 *nix 文件系统的 root.
{{</admonition>}}

本博客使用的是 even theme. 但是俺相信参照本文在其他博客中进行实现也会大差不差。

俺在自己实现了此种风格之后即向 [hugo-theme-even](https://github.com/olOwOlo/hugo-theme-even) 提出了 PR，even theme 的用户应该可以在不久以后在新版本中得到 `mermaid` 支持。

### 导入 mermaid.min.js

实现应当遵循原理的指示精神，首先咱们需要拥有 `mermaid` 库。通过观察源码猜测 even 主题在 `/static/layouts/partials/scripts.html` 中进行 JS 的载入。于是咱们向其中插入能够引入 `mermaid` 模块的 script 标签。（实际上只要是会被加载的 HTML 文件都可以插）

这个标签在 HTML 文件里随便找个地方插就完事了，如果完全不懂，建议怼到 HTML 文件末尾。

#### 通过 CDN 导入

为了保持文章的时效性，俺不会直接提供该 script 标签，但是你可以在 [jsdelivr.com](https://www.jsdelivr.com/package/npm/mermaid?path=dist) 中找到你中意版本的 `mermaid.min.js` 并按下右边的 Copy HTML + SRI 按钮即可获得对应的标签。

#### 通过本地文件导入

由于房间里的大象太大挡住了境外CDN的信号，所以你也可以选择本地文件。在网上找到你中意版本的 `mermaid.min.js`, 下载下来并放在 `/static/lib/mermaid/` 目录下（没有就新建一个），这时可以得到对应的 script 标签如下：

```html
<script src="{{ "lib/mermaid/mermaid.min.js" | relURL }}"  crossorigin="anonymous"></script>
```

本人对境外 CDN 在内地的稳定性持悲观态度，建议中国大陆用户使用此方法。

### 替换 HTML 标签

观察 even 主题的源码，咱们可以在 `/assets/js/` 目录下发现 `main.js` 和 `even.js`。在 `even.js` 中插入如下代码段：

```javascript

// 插到文件末尾完事了

Even.mermaid = function () {
  if (!window.mermaid) return;

  const blocks = document.querySelectorAll("pre code.language-mermaid");

  // change all <pre><code mamaid> tags to <div class="mermaid">
  // mermaid will handle the rest!
  for (let i = 0; i < blocks.length; i++) {
    if (!window.hljs && i % 2 === 0) continue;

    const block = blocks[i];
    const rootElement = window.hljs
      ? block.parentElement
      : block.parentElement.parentElement.parentElement.parentElement
          .parentElement.parentElement.parentElement;

    var container = document.createElement("div");
    const id = `js-mermaid-diagrams-${i}`;
    container.id = id;
    container.className = "align-center mermaid";
    container.textContent = block.childNodes[0].textContent;
    rootElement.parentElement.replaceChild(container, rootElement);
  }

  window.mermaid.mermaidAPI.initialize(
    window.mermaidOptions
      ? window.mermaidOptions
      : { securityLevel: "loose", theme: "base" }
      // 替换它可以修改 mermaid 的样式=======↑
 );
};
```

在 `main.js` 中插入：

```javascript
// 个人比较喜欢插在 Even.sequence(); 的后面
Even.mermaid();
```

现在你应该可以在自己的博客中写 `mermaid` 了。

---

### 工具推荐

- [Typora](https://typora.io)

  写 MarkDown 咋能少得了 「[Typora](https://typora.io/)」？

- [Visual Studio Code](https://code.visualstudio.com/)

  俺写这篇文章用的无所不能的编辑器，建议添加插件：

  - Markdown All in One
  - Markdown Preview Enhanced
  - Mermaid Markdown Syntax Highlighting

  之所以用 Visual Studio Code，很大程度上是因为它的 Vim 拓展。 “2021年了不会还有人不会用 Vim 吧？”

  - vim

