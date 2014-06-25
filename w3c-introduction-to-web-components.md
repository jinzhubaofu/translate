# W3C 网页组件导言

翻译自[W3C Introduction to Web Components](http://www.w3.org/TR/2013/WD-components-intro-20130606/)

## 2 引言

网页组件模型(Web Components)由五个部分组成：

1. [模板](http://www.w3.org/TR/components-intro/#template-section)，定义了一块惰性标签，但是可以随后激活使用。
2. [装饰器](http://www.w3.org/TR/components-intro/#decorator-section)，定义了如保基于CSS选择器来应用模板，来现实丰富的视觉表现和文档改变行为
3. [自定义元素](http://www.w3.org/TR/components-intro/#custom-element-section)，允许开发人员使用新的标签名和脚本接口，来定义自己的元素。
4. [影子DOM](http://www.w3.org/TR/components-intro/#shadow-dom-section)，封装了一个DOM子树，用来完成更加可依赖的用户界面模块 。
5. [引入](http://www.w3.org/TR/components-intro/#imports-section)，定义如何把模板、装饰器和自定义元素作为资源打包和加载的。

每个单独部分都是有用的。组合在一起使用，网页组件可以让网页开发者来定义新的部件，拥有只用CSS无法完成的丰富视觉展现和交互效果，以及现有的脚本库无法提供的在结构与重用上的易用性。

本文讨论了每个部分内容。每一节最后的部分都会说明开发者怎样结合起来使用它们。

## 3 模板

[HTML模板规范](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/templates/index.html)是网页组件模板部分的正式描述。

`<template>`元素包含了一块将来才会使用的标签。 `<template>`元素的内容由解析器解析，但是是惰性的：其中的脚本不会执行，图片也不被下载，以此类推。`<template>`元素是不会被渲染的。

`<template>`元素有一个`content`属性，它用一个文档片`document fragment`来保存了模板的内容。当开发者想要使用这些内容时，他们从这人属性里可以移动或者拷贝这些结点。

```html
<template id="commentTemplate">
    <div>
        <img src="">
        <div class="comment-text"></div>
    </div>
</template>
<script>
function addComment(imageUrl, text) {
  var t = document.querySelector("#commentTemplate");
  var comment = t.content.cloneNode(true);
  // Populate content.
  comment.querySelector('img').src = imageUrl;
  comment.querySelector('.comment-text').textContent = text;
  document.body.appendChild(comment);
}
</script>
```

因为在模板元素中的内容并不在文档中，而在文档片中，所以可以通过将模板内容的拷贝添加到文档中来使它*活*起来，此时其中的脚本就会执行，图片就会加载。 例如，模板可以被用来直接在文档中内置很大的脚本，但在它们真正被使用之前，不会有解析它们的消耗。

## 4 装饰器

装饰器与网页组件的其他部分不同，暂时还没有规范。 这一章节里我们简单描绘装饰器所追求的概念。这一部分并没有正式的规范。

`装饰器`是一个用来增强或覆盖某个已有元素的展现形式的东西。与所有的展现方式一样，装饰器的样式也是由CSS来控件的。但是能够用标签来添加额外的特殊样式是装饰器的独门特技。

`<decorator>`元素包含了一个`<template>`元素，指定了用来渲染装饰器的标签。

```html
<decorator id="details-open">
    <template>
        <a id="summary">
          &blacktriangledown;
          <content select="summary"></content>
        </a>
        <content></content>
    </template>
</decorator>
```

`<content>`元素指定了装饰器元素内容（装饰器的子元素）在渲染中所放置的位置。

`decorator`CSS样式会被应用在装饰器上：

```css
details[open] {
    decorator: url(#details-open);
}
```

装饰器和样式会使下会的标签：

```html
<details open>
    <summary>Timepieces</summary>
    <ul>
      <li>Sundial
      <li>Cuckoo clock
      <li>Wristwatch
    </ul>
</details>
```

被渲染成这样：

![decoratoer-rednered](http://ludafa-gist.qiniudn.com/decorator-rendered.jpg)

即使`decorator`的样式属性可以指向网页中的任何资源，但是装饰器不会应用这些样式，除非这些样式被页面文档加载。 生成展现的标签被限制只能用于展现：它永远不能执行脚本（包括行内事件处理器）并且不能被[编辑](http://www.whatwg.org/specs/web-apps/current-work/multipage/editing.html#editing-0)

### 4.1 装饰器的事件

装饰器也可以绑定事件处理函数来实现交互。

因为装饰器是易变的，所以给装饰器模板中的结点绑定事件监听器或者依赖它们的任何状态并没有什么用，不管装饰器有没有绑定事件监听，这些结点都被消灭掉。装饰器事件是通过一个事件控件器来实现的。

![Fig. Event handler registration](http://www.w3.org/TR/components-intro/event-handler-registration.png)

为了使用事件控制器来注册事件监听器，模板引入了一个`script`元素。这段脚本在`decorator`元素被插入文档或者当作外部文档的一部分被加载时会执行一次。这段脚本必须是一个注册事件的数组：

```html
<decorator id="details-open">
    <script>
        function clicked(event) {
            event.target.removeAttribute('open');
        }
        [{selector: '#summary', type: 'click', handler: clicked}];
    </script>
    <template>
        <a id="summary">
        <!-- as illustrated above -->
```

事件控件器解析这个数组，并对添加了事件处理器函数的结点所产生的事件做路由转发。

![Fig. Event routing and retargeting](http://www.w3.org/TR/components-intro/event-routing-retargeting.png)

当事件监听函数被调用，事件的目标对象是装饰器所应用的内容，而不是模板中的内容。在上面的例子里，点击▾（在模板中定义的）所产生的事件会被转发到`clicked`函数（这个函数被注册到`#summary`元素上，同样也是在模板中定义的），但是`event.target`属性指则向被装饰的`<details>`元素。这个注册是必需的，因为装饰器指定了展现；但不影响文档中的DOM结构。

通过移除`open`属性，装饰器就不再会应用于原有的元素了，因为带有`decorator`属性的样式选择器不再匹配了。元素取消应用装饰器，会返回到装饰前的状态。然而开发者可以写另外一个装饰器来完成"关闭"状态的渲染效果，以这种方式来实现无状态交互只需要触发不同的装饰器就可以了。

```html
<style>
details {
    decorator: url(#details-closed);
}
details[open] {
    decorator: url(#details-open);
}
</style>

<decorator id="details-closed">
    <script>
        function clicked(event) {
            event.target.setAttribute('open', 'open');
        }
        [{selector: '#summary', type: 'click', handler: clicked}];
    </script>
    <template>
        <a id="summary">
            &blacktriangleright; <content select="summary"></content>
        </a>
    </template>
</decorator>

<decorator id="details-open">
    <!-- as illustrated above -->
```

这里使用了两个装饰器。一个代表了详情视图的关闭状态，另一个代表着详情视图的展开状态。每个装饰器使用了一个事件处理器，切换元素的展开状态来响应点击事件。内容元素的`select`属性会在接下来的内容里详情介绍。

## 5 自定义元素

[自定义元素规范](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/custom/index.html)是这一部分内容的正式规范。

自定义元素是可以由开发者来定义的新类型DOM元素。与装饰器的无状态和易变性不同，自定义元素可以封装状态并提供脚本接口。下面的表格总结了装饰器和自定义元素之间的关键区别。

|  | 装饰器 | 自定义元素 |
|--------|--------|--------|
|生命周期|易变的，当一个样式选择器匹配时才会生效|稳定的，匹配元素的整个生命周期|
|动态应用和取消应用|是，基于样式选择器|否，在元素创建时就固定了|
|是否可以通过脚本访问|否，透明DOM；不能添加接口|是，可以通过DOM访问；可以提供接口|
|状态|无状态保护|有状态DOM对象|
|交互行为|通过改变装饰器来模拟交互行为|主要使用脚本和事件来完成交互行为|

### 5.1 定义一个自定义元素

`<element>`元素定义一个自定义元素。 它指定了元素的类型，并通过使用`extends`属性来细化类型：

```html
<element extends="button" name="fancy-button">
    …
</element>
```

`extends`属性指定了自定义元素所扩展的标签名称。自定义元素的实例会拥有这里所指定的标签名称。

`name`属性指定了自定义元素的名称，这里指的是在标记中所使用的名称。名称必须使用连字符来连接单词。

由于不是所有客户端都支持自定义元素，开发者应当扩展那些与新元素最贴近的HTML元素。比如，如果想要定义一个新元素，用来响应点击事件做相应的动作，那么应该扩展`button`元素。

当没有一个HTML元素在语义上与自定义元素相近，开发者可以省略`extends`属性。这些元素会使用`name`属性的值作为标签名称。因此，这种元素被称为自定义标签。不支持自定义元素的客户端会按无法区分语义的[HTMLUnknownElement](http://www.whatwg.org/specs/web-apps/current-work/multipage/elements.html#htmlunknownelement)来处理。

### 5.2 方法与属性

通过添加一个内置的`<script>`元素，你可以给自定义元素的原型对象添加方法和属性，来定义一个自定义元素的脚本API。

```html
<element name="tick-tock-clock">
  <script>
    ({
      tick: function () {
        …
      }
    });
  </script>
</element>
```

这段脚本的最终值所包含的这些属性被会拷贝到一个原型对象上。在上面的例子中，`<tick-tock-clock>`元素会有一个`tick`的方法。

### 5.3 生命周期回调

自定义元素有生命周期回调，可以利用这些回调来设定自定义元素的展现。它们是：`readyCallback`，当自定义元素被创建后被调用；`insertedCallback`，当自定义元素被插入到页面文档时被调用；`removedCallback`，当自定义元素被从文档中移除时调用。

下面的例子阐释了如何使用模板、影子DOM（[将会在后面的章节了详细描述](http://www.w3.org/TR/components-intro/#shadow-dom-section)）以及生命周期回调来创建一个时钟元素：

```html
<element name="tick-tock-clock">
  <template>
    <span id="hh"></span>
    <span id="sep">:</span>
    <span id="mm"></span>
  </template>
  <script>
    var template = document.currentScript.parentNode.querySelector('template');

    function start() {
      this.tick();
      this._interval = window.setInterval(this.tick.bind(this), 1000);
    }
    function stop() {
      window.clearInterval(this._interval);
    }
    function fmt(n) {
      return (n < 10 ? '0' : '') + n;
    }

    ({
      readyCallback: function () {
        this._root = this.createShadowRoot();
        this._root.appendChild(template.content.cloneNode());
        if (this.parentElement) {
          start.call(this);
        }
      },
      insertedCallback: start,
      removedCallback: stop,
      tick: function () {
        var now = new Date();
        this._root.querySelector('hh').textContent = fmt(now.getHours());
        this._root.querySelector('sep').style.visibility =
            now.getSeconds() % 2 ? 'visible' : 'hidden';
        this._root.querySelector('mm').textContent = fmt(now.getMinutes());
      },
      chime: function () { … }
    });
  </script>
</element>
```

### 5.4 在标记中使用自定义元素

由于自定义元素使用现存的HTML标签名称——`div`, `button`, `option`等等——所以我们需要使用一个属性来指定何时开发者想要使用一个自定义标签。这个属性的名字是`is`，它的值是自定义元素的名称。例如：

```html
<element extends="button" name="fancy-button">  <!-- definition -->
    …
</element>

<button is="fancy-button">  <!-- use -->
    Do something fancy
</button>
```

### 5.5 在脚本中使用自定义元素

作为`<element>`元素的替代，你也可以使用`register`方法来注册自定义元素。这种情况下你可以通过直接操纵原型对象来设定元素的方法和属性。`register`返回一个函数，可以调用它来创建一个自定义元素的实例：

```javascript
var p = Object.create(HTMLButtonElement.prototype, {});
p.dazzle = function () { … };
var FancyButton = document.register('button', 'fancy-button', {prototype: p});
var b = new FancyButton();
document.body.appendChild(b);
b.addEventListener('click', function (event) {
    event.target.dazzle();
});
```
通过`<element>`或者`<register>`定义的自定义元素可以通过标准的`createElement`方法来实例化：

```javascript
var b = document.createElement('button', 'fancy-button');
alert(b.outerHTML); // will display '<button is="fancy-button"></button>'
var c = document.createElement('tick-tock-clock');
alert(c.outerHTML); // will display '<tick-tock-clock></tick-tock-clock>'
```

### 5.6 元素升级

当自定义元素的定义被加载，每个与定义相匹配的元素会被升级。这个升级过程更新元素的原型，这可以使脚本可以使用元素的API，并且激活生命周期回调。

你可使用样式伪类`:unresolved`来避免没有样式内容的闪动。这个伪类会匹配任意还没有可用定义的元素：

```html
<style>
tick-tock-clock:unresolved {
  content: '??:??';  
}
</style>
<tick-tock-clock></tick-tock-clock> <!-- will show ??:?? -->
```

这个伪类也能够让脚本来阻止未升级元素的交互行为：

```javascript
// Chime ALL the clocks!
Array.prototype.forEach.call(
  document.querySelectorAll('tick-tock-clock:not(:unresolved)'),
  function (clock) { clock.chime(); });
```

一个元素如果想要通过页面中其他元素自己被已经升级了，可以释放一个自定义的事件来完成。如果脚本想要在元素完成升级前延迟交互行为就可以监听这个事件。

### 5.7 扩展自定义元素

自定义元素作为HTML元素的扩展，你当然也可以扩展自定义元素。你可以将`<element>`元素`extends`属性值设定为自定义元素的名称，或者包括另一个自定义元素的原型链。

```html
<element extends="tick-tock-clock" name="grand-father-clock">
    …
</element>
<script>
var p = Object.create(Object.getPrototypeOf(document.createElement('tick-tock-clock')));
p.popOutBirdie = function () { … }
var CuckooClock = document.register('cuckoo-clock', {prototype: p});
var c = new CuckooClock();
c.tick();         // inherited from tick-tock-clock
c.popOutBirdie(); // specific to cuckoo-clock
</script>
```

## 6 影子DOM

[影子DOM规范](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html)是这一部分的正式描述。

影子DOM是DOM树结点的附属。这些影子DOM子树可以与一个元素相关联，但是不能以元素子结点出现。相反的，这些子树形成了自己的作用域。例如，一个影子DOM子树可以包含ID和样式，并与页面文档中的ID和样式相重叠，但由于影子DOM子树与文档相分离，这些影子DOM子树中的ID和样式并不会文档中的相冲突。

通过调用[createShadowRoot](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#api-partial-element-create-shadow-root)方法，影子DOM可以应用到一个元素上，并返回一个[ShadowRoot](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#shadow-root-object)结点，接着你可以将这个结点就填充到DOM结点上。

![shadow-dom](http://ludafa-gist.qiniudn.com/shadow-dom.jpg)

一个带有影子DOM的元素被称为[影子主](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-shadow-host)。当一个元素拥有影子DOM，这个元素的子结点就不会被渲染；影子DOM的内容取而代之，被渲染出来。

![shadow-render](http://ludafa-gist.qiniudn.com/shadow-render.jpg)

### 6.1 插入点

一个影子DOM子树可以在渲染结果中使用一个[content](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#content-element)元素来指定一个[插入点](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-insertion-point)。影子主的子结点会被显示在插入点。`<content>`元素只表现为一个渲染的插入点——它不会改元素在DOM中的位置。

在影子DOM中你可以拥有多个插入点！[select](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#markup-content-select)属性使你可以选择哪些子结点出现在哪个插入点，正如[details-open decorator example](http://www.w3.org/TR/components-intro/#example-details-open-decorator)所展示的：

```html
<a id="#summary">
  &blacktriangledown;
  <content select="summary"></content>
</a>
<content></content>
```

插入点使你可以重新排序或者省略掉影子主子结点的渲染，却不会引起多次渲染。在选择子结点时，树顺序决定了每个元素是否能够通过。一旦子结点被选中在插入点中渲染，它不能再一次被声明，这也就是为什么`detail-open`装饰器只渲染一次总结。

### 6.2 重投影

在影子DOM子树中的元素可以拥有它们自己的影子根结点。当这种情况出现时，嵌套的影子DOM子树的插入点作用在外层影子DOM子树中的插入点上。如下例子所示：

```html
<!-- document -->
<div id="news">
  <h1>Good day for kittens</h1>
  <div class="breaking">Kitten rescued from tree</div>
  <div>Area kitten "adorable"&mdash;owner</div>
  <div class="breaking">Jiggled piece of yarn derails kitten kongress</div>
</div>

<!-- #news' shadow -->
<template id="t">
  <content select="h1"></content>
  <div id="ticker">
    <content id="stories"></content>
  </div>
</template>

<!-- #ticker's shadow -->
<template id="u">
  <content class="highlight" select=".breaking"></content>
  <content></content>
</template>

<script>
// Set up shadow DOM
var news = document.querySelector('#news');
var r = news.createShadowRoot();
var t = document.querySelector('#t');
r.appendChild(t.content.cloneNode(true));

var ticker = r.querySelector('#ticker');
var s = ticker.createShadowRoot();
var u = document.querySelector('#u');
s.appendChild(u.content.cloneNode(true));
</script>
```

从概念上讲，第一个文档和第一个影子根结点结合，形成了下边的虚拟DOM树。插入点已经被抹去了，影子主元素的子结点出现它们的位置上：

```html
<div id="news">
  <h1>Good day for kittens</h1>
  <div id="ticker">
    <div class="breaking">Kitten rescued from tree</div>
    <div>Area kitten "adorable"&mdash;owner</div>
    <div class="breaking">Jiggled piece of yarn derails kitten kongress</div>
  </div>
</div>
```

接下来，这个中间虚拟树与第二个影子根结点结合，形成了另一个虚拟树。所有的breaking news现在都是在最上边。这是因为`<content class="highlight" select=".breaking">`插入点。注意元素被重新排序了，尽管`breaking`样式并没有出现在`#ticker`元素的子结点里。这是因为插入点从中间虚拟树中选择结点，而不是原有树。以这种方式，通过多重插入点的方法来展现一个结点叫作重投影。

```html
<div id="news">
  <h1>Good day for kittens</h1>
  <div id="ticker">
    <div class="breaking">Kitten rescued from tree</div>
    <div class="breaking">Jiggled piece of yarn derails kitten kongress</div>
    <div>Area kitten "adorable"&mdash;owner</div>
  </div>
</div>
```

### 6.3 后备内容

一个插入点可能拥有会内容。这种插入点被称为[后备内容](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-fallback-content)，因为只有当被有内容被分发到插入点里时，这些内容才会被展现。例如，如果heading和/或sories不可用，news ticker影子结点就可能被修改来显示默认文本：

```html
<!-- #news' shadow -->
<template id="t">
  <content select="h1">Today's top headlines</content>
  <div id="ticker">
    <content id="stories">
      No news
      <button onclick="window.location.reload(true);">Reload</button>
    </content>
  </div>
</template>
```

### 6.4 多个影子子树

任何元素都可以拥有多个影子DOM子树。不要这么迷惑！事实上，这很常见。当你扩展一个自定义元素时，它已有一个影子DOM子树了。那个可怜的老树会发生什么呢？我们可以简单地不再碰这个树栖的烫手山芋，但是如果你不想这么干呢？如果你想重用一下它？

还有另外一种插入点来完成这个目标：[shadow](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#shadow-element)元素，拉取上一次应用的影子DOM子树（也被称作[老树](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-older-tree)）。例如，这有一个自定义元素，扩展了[tick-tock-clock](http://www.w3.org/TR/components-intro/#example-tick-tock-clock)元素，并添加了一个方向指示符：

```html
<element name="sailing-watch" extends="tick-tock-clock">
  <template>
    <shadow></shadow>
    <div id="compass">N</div>
  </template>
  <script>
    …
  </script>
</element>
```

既然一个元素可以拥有多个影子，我们就需要明白这些影子是怎么彼此交互的，以及这些交互在渲染元素子结点上有哪些影响。

首先，影子DOM子树的应用顺序是非常重要的。因为你不能移除一个影子根结点。顺序是这样的：


1. 客户端的影子DOM（哈哈，我们最先到！）
2. 基础自定义元素的影子DOM
3. 最先得到的自定义元素的影子DOM
4. ...
5. 通过脚本添加的专用影子
6. 装饰器影子（应用和移除样式规则——技术上讲不是影子DOM，但是它的插入点与影子DOM类似）

接下来，我们拿到影子DOM子树的[栈](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-tree-stack)，并反向遍历它，从最后一个应用的（最年轻的）子树开始。每个`<content>`插入点，就进入到更老的树，获取到影子主的子结点，一般来讲都会需要使用到它们。

这里开事情变得有意思了。一但我们对子结点做洗牌，把它们放到它们正确的位置上去，我们检查一下是否有一个`<shadow>`元素。如果没找到，就结束了。

如果我们找了，我们扑通一下把列表里下一个影子子树放到`<shadow>`元素的位置上，然后重复替换`<content>`插入点和替换`<shadow>`，直到我们到达栈的最后。

![](http://ludafa-gist.qiniudn.com/multiple-shadow-dom.png)

有一种简单的方法来记忆这种工作方式：

+ 最近应用的影子DOM子树的`<content>`插入点内有最新鲜的子结点
+ 如果一旦能够找到下一个最近的`<shadow>`元素，那么就通过它来搜寻剩余的子结点
+ 循环往复这样操作，直到当前的影子DOM子树没有`<shadow>`元素或者我们已经处理了最老的DOM子树。

### 6.5 CSS与影子DOM

当构建一个自定义元素时，很自然地会思考标记（属性、内容）与脚本接口。通常情况，样式与页面之间的交互也是同等的重要。 影子DOM为开发者提供了很多办法来控制影子DOM的内容与样式的交互。

影子DOM子树被一个看不见的[边界](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-shadow-boundary)所包裹。这层包裹通常应用了客户端提供的样式，而不是开发者的。然而继承仍然可以工作。一般情况下这就是你所期待的：在上边提到的[sailing-watch](http://www.w3.org/TR/components-intro/#example-sailing-watch)例子中，如果页面中设定了样式，文本被漂亮的海绿色包裹，整体保持了一个航海主题风格。在影子DOM中的文本（例如那个方向指示符"N"）同样也是海绿色，因为`color`属性是一个可继承属性。但是如果开发者设定`<div>`元素有一个橙色的边框样式，那么方向指示符不会有橙色的边框，因为`border`属性不是可继承属性。

一个影子根点结有两个属性来控制这种行为。第一个，`applyAuthorStyles`，可以指定使用开发者的样式。设定这个属是合理的，因为当写一个影子DOM时，你需要让它的展现效果尽可能地贴近周围的内容。有一个警告：CSS选择器是会匹配影子DOM子树的，也不是原来的扁平树。使用这个属性时要注意那些子孙、兄弟和"nth-of-x"选择器。

第二个控制了影子DOM子树的根结点属性是`resetStyleInhertance`。如果这个属性被设为`true`，那么所有影子结点边界的所有可继承样式属性都被重置为[初始值](http://www.w3.org/TR/CSS21/about.html#initial-value)。

如果`applyAuthorStyles`是`false`并且`resetStyleInheritance`是`true`，那么你就会得到一个干净的板面。你的元素与页面中的样式是绝缘的——即使是继承忏悔——并且你可以还可以使用一个浏览器重置样式来构建你想要的样式。

在插入点也有一个相似的边界。影子DOM子树的样式不能应用到那些被分配到插入点内影子主元素的子结点上。然而，一些插入点，特别是那么指定了特殊选择器和用途的结点，需要给被分配内容应用样式。影子DOM设置了一个[伪类::distributed](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#distributed-pseudo-element)来达成这个目的。我们用这个新例子来详细阐述：

```html
<!-- #ticker's shadow -->
<template id="u">
  <style>
    content::distributed(*) {
      display: inline-block;
    }
    *::distributed(.breaking) {
      text-shadow: 0 0 0.2em maroon;
      color: orange;
    }
  </style>
  <content class="highlight" select=".breaking"></content>
  <content></content>
</template>
```
这个例子中有两个使用`::distributed`的规则。其中左边的这个选择器应用到影子DOM子树；这个选择器中圆括号里的样式会应用到被分配到插入点里的元素上。所以第一个选择器`content:distributed(*)`应用到所有被分配到`<content>`插入点里的元素；同时，第二个选择器`*::distributed(.breaking)`则会应用到所有被分配到插入点内并带有`breaking`类样式的元素上，给它们一个健康的粟色发光效果。使用选择器`.highlight::distributed(*)`作为替代，可以减少选择到冗余元素。

影子DOM也可以给它的主元素添加样式。影子DOM给主元素添加样式是很自然的想法，因为主元素是组件的最外层部分。例如，为了使ticker用一个滚动展现效果来替代静态效果，我们可以在[@host规则](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#host-at-rule)里定义：

```html
<!-- #ticker's shadow -->
<template id="u">
  <style>
    @host {
      :scope {
        white-space: nowrap;
        overflow-style: marquee-line;
        overflow-x: marquee;
      }
    }
    content::distributed(*) {
      display: inline-block;
    }
    *::distributed(.breaking) {
      text-shadow: 0 0 0.2em maroon;
      color: orange;
    }
  </style>
  <content class="highlight" select=".breaking"></content>
  <content></content>
</template>
```

最后，有两种方法来允许页面来控制影子DOM子树内容的样式。第一方法，通过给影子DOM子树里的一个指定元添加一个伪ID。然后就可以把它当作一个伪元素，把开发者的样式就来指向它。例如，外层的"news"影子DOM可能想让开发者来指定ticker部分的样式。那么可以通过设定[pesudo属性](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#custom-pseudo-elements)来搞定。接着，开发者的样式就可以定位到组件中的这一部分了。

```html
<script>
// Set up shadow DOM
…
var ticker = r.querySelector('#ticker');
ticker.pseudo = 'x-ticker';
…
</script>

<!-- change the appearance of the ticker part -->
<style>
#news::x-ticker {
  background: gray;
  color: lightblue;
}
</style>
```

另外一种让页面有选择性给影子DOM内容添加样式的方法是使用CSS变量。比如，除了硬编码橙色带粟色之外，我们还可以编写一对高亮颜色：

```html
<!-- #ticker's shadow -->
<template id="u">
  <style>
    @host {
      :scope {
        white-space: nowrap;
        overflow-style: marquee-line;
        overflow-x: marquee;
      }
    }
    content::distributed(*) {
      display: inline-block;
    }
    *::distributed(.breaking) {
      text-shadow: 0 0 0.2em var(highlight-accent, maroon);
      color: var(highlight-primary, orange);
    }
  </style>
  <content class="highlight" select=".breaking"></content>
  <content></content>
</template>

<!-- change the appearance of the ticker part -->
<style>
#news::x-ticker {
  background: gray;
  color: lightblue;
  var-highlight-primary: green;
  var-highlight-accent: yellow;
}
</style>
```

伪元素有助于开放指定元素所有属性给开发者来编写样式。而CSS变量则有助于允许开发者给一小部分属性重新设定样式，诸如主题颜色就可以多次地重用了。

### 6.6 影子DOM的事件

为了保证影子DOM子树中的元素不暴露给外界，在释放子树内部事件时我们需要做一些必要的工作。

首先，有些事件（像`mutation`事件和`selectstart`事件）会被完全阻止逃出影子DOM子树——它们永远不会被外界监听到。

那些能够逃出[影子DOM边界](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#dfn-shadow-boundary)的事件是需要被[注册](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#event-retargeting)的——它们的`target`和`relatedTarget`值会被修改指向影子DOM子树的主结点。

在一些情况下，像`DOMFocusIn`,`mouseover`,`mouseout`事件需要额外的注意：如果你在影子DOM子树内部的两个元素之间移动鼠标，你并不希望把这些垃圾事件发送给页面文档，因为这些事件在修改了目标元素之后看上去非常的无厘头。（神马？这个元素刚刚汇报说鼠标从自己移动到了自己？！）

## 7 引入

[The HTML Imports specification](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/imports/index.html)是这一部分的正式规范。

可使用下边的标签从外部文件加载自定义元素和装饰器：

```html
<link rel="import" href="goodies.html">
```

只有装饰器元素和`<element>`元素才会被客户端解释，其他的元素则被忽略。但是通过`import`属性引入的DOM文档还是可以给脚本使用的。跨域加载的文档使用CORs来决定它们是不是可以跨域运行的。


