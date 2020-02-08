---
title: "Web：HTML 常用标签"
author: "Chenghao Zheng"
tags: ["Web"]
categories: ["Study notes"]
date: 2020-02-08T13:19:47+01:00
draft: false
---

HTML（Hyper Text Markup Language）超文本标记语言，是构建网站的基石。游览器通过解析 HTML 来将互联网中传输的 `字节流` 渲染成丰富多彩的可视化网页。

由于我不是写前端的，就用在线 Web 调试工具 [饥人谷 JsBin](http://js.jirengu.com/?html,output) 和 [代码沙盒](https://codesandbox.io) 来进行 HTML 的编辑。

# HTML 章节与内容标签

### 1. HTML 起手式

HTML 代码由许许多多不同的标签（tag）构成。下图为 HTML 的一个常规模版：

![](/images/HTML起手式.png)

* `<head>` 里面一般放一些看不见的元素，如编码方式、全文样式 `<style>` 、标题、JS等
* `<body>` 里面写网页的具体内容
* 模版中的内容只需要改 title 和语言，其他暂时都不变。
* 有个输入标签的快捷键，打出标签单词后按键盘上的 `Tab` 键就可以快速补全代码了。

### 2. HTML 章节标签

章节标签用于划分网页 / 文章的层级，使页面结构清晰，也便于添加 CSS 样式

1. 标题：`<h1~h6>` 字体从大到小，h1 一个页面最多一个，不能为了调整字体大小而使用
2. 章节：`<section>` 开始了一个新的章节，里面一般用 h2，h3；章节里面可以再套章节
3. 文章：`<article>`
4. 段落：`<p>`
5. 头部：`<header>` 放置顶的广告
6. 脚部：`<footer>` 最下面的版权声明等，`&copy;`打出版权的符号 
7. 主要内容：`<main>` 里面是标题+章节
8. 旁支内容：`<aside>` 可以写导航，或在 main 的下面写个参考资料
9. 划分：`<div>` 用于划分页面结构，使结构更加清晰

![](/images/章节标签.png)

### 3. 全局属性

全局属性一般写在章节标签中，为一个块添加功能或者更改样式：

1. `class`  给标签分类，只是个标记，可以写多个 class 名称，方便之后设置样式
2. `contenteditable`  使元素可以被用户编辑，可以做一个自己的编辑器
3. `hidden`  写在标签字的后面，将对应的整个块 block 隐藏起来 
4. `id` 
   1. 用 id 还是用 class，如果当前标签内容是全页面唯一的，用 id；不然用 class
   2. 不到万不得已，千万不要用 id，因为 id 不报错...
   3. `#xxx{}` / `[id=xxx]{}` 写 id 相关的样式
   4. id 可以直接在 JS 中调用，`xxx.style.border = '1px solid green'` 当有重复 id 时不生效
   5. id 名字不可与 windows 自带的全局属性重复。
5. `<style>`  写在head里面，
   1. `[class=middle bordered]` 等号判断需要 class 名完全相等，不推荐使用
   2.  `.middle` 可以匹配 class 名中用空格分配的一部分
   3. style 可以放 body 然后改 display: block; 显示出来 ---> 加 contenteditable，就可以让用户自己改自己的样式
   4. style 也可以直接写在章节标签后面 `style="border: 5px solid green;"`，这里的优先级比写在 head 中的要高 （JS 的优先级高于章节标签，JS 会覆盖它们其他两个）
6. `tabindex`
   1. 让没有鼠标的用户用键盘上的 tab 也可以实现网页的游览
   2. 加在章节标签后面  tabindex=1 2 3 ... ，来控制 tab的顺序的；0 是最后一个，-1 不要到这里
7. `title`
   1. 标题内容太多，一行放不下，然后不换行，最后....  
   2. 用户鼠标放上到...后，显示 title 的值，可以显示完整的内容。

![](/images/全局属性.png)

### 4. 内容标签

1. ol + li：ordered list；list item
2. ul + li：unordered list  
3. dl + dt + dd：description list；description term；description data
4. pre   
   1. html  默认多个空格 / 换行会缩成一个空格 ---> 因为写代码有时候要换行缩进
   2. 用 pre 标签包住想要保留空格 / 回车的内容就可以了，一般用在 code 中。
   3. 一般会在 style 中声明 pre 的字体：`font: inherit;`
5. hr：水平分割线
6. br：换行，直接写在要换行的地方就可以了
7. a ：超链接，href="qq.com" -  target="_blank" 打开一个新的窗口 可以安装个插件 Death To _blank 来实现自己控制是否打开新的标签
8. em：emphasis 强调 - 默认斜体，可以自定义样式，指语气很重要
9. strong：重要 - 默认加粗，可以自定义样式，指内容很重要
10. code
    1. code 里面的字是等宽的，方便对齐
    2. 默认为内联 inline，外面套 pre 就可以换行了
11. quote：内联的引用
12. blockquote：块级的引用
13. iframe：内嵌窗口（套娃），已经很少使用了，还有些老系统在用


```html
前端的日常工作有：
<ol>
    <li>写页面</li>
    <li>发邮件</li>
    <li>和产品经理探讨人生</li>
</ol>
老王的兴趣爱好有：
<ul>
    <li>唱</li>
    <li>跳</li>
    <li>篮球</li>
</ul>
```

### 5. 文本样式

```html
斜体：<i></i>
粗体：<b></b>
删除线：<s></s>
下划线：<u></u>
上标：<sup></sup>
下标：<sub></sub>
```

# HTML 常用重点标签

重点介绍 a、table、img、form 这几个标签。

### 1. a 标签

a 标签的作用：

* 跳转外部页面
* 跳转内部锚点
* 跳转到邮箱或电话等

#### 1.1 href 属性

1. href (hyper ref)  超链接
   1. `<a href="https://google.com">超链接</a>`
   2. 像用户一样打开网页：使用 `http-server -c-1`（不要缓存），在结尾加上路径
   6. 方法二：yarn global add parcel  --> parcel a-href.html （直接接 html 文件名）
2. 网址
   1. `//google.com`  无协议地址，它会继承当前的 http ---> 跳转到 google 真正的地址
   2. // 最高级，它会自动选择合适的 http / https，以后写网址都写这个
   3. 检查 - Network - Preserve.log 勾选 - 点击超链接 - 查看网页跳转记录
3. 路径
   1. /a/b/c.html    http 的相对路径是相对于 http 服务的根目录
   3. index.html  等价于 ./index.html 在当前目录下寻找
4. 伪协议
   1. `<a href="javascript:alert(1);"` 点击这个标签就会执行 JS 代码 - 弹窗，显示 1 （早期的用法，现在不怎么用了）
   2. href="javascript:;"  链接点击之后，什么都不做
   3. href="" 留空 ---> 点击链接页面会刷新
   4. href="#" ---> 页面自动滚到顶部
   5. href="#xxx" 自动跳转到 [id=xxx] 的标签，做锚点。
   6. href="mailto:zch693922442@outlook.com"    发邮件
   7. href="tel:1234"  直接打电话。

#### 1.2 target 属性

1. target=_blank  指定在哪个窗口打开链接
2. 取值
   1. _blank：新窗口打开
   2. _top：在最顶层打开链接，从 iframe 跳至外面
   3. _parent：父级窗口打开
   4. _self：是默认值，在当前页面打开（google 不允许别人用 iframe 打开它）
   5. target="xxx" 如果有个 xxx 的窗口就用它打开链接，没有就创建一个
   6. target=iframeName  网页内套娃

#### 1.3 其他属性

1. download
   1. download 下载这个网页，很多游览器不支持这个
2. rel=noopener
   1. 防止一个 bug，学了 JS 之后再讲

### 2. table 标签

table 标签里面只能有三个标签 thead、tfoot、tbody  （相互顺序无影响）

![](/images/表格标签.png)

* tr=table row；th=table head；td=table data；（没写默认为 td）
* 如果忘记写 thead tfoot 会在网页内自动设置被 tbody 
* 如果想给表该样式：需要使用 CSS
* 相关的样式
  1. table-layout：fixed 等宽；auto 表格宽度随着内容改变
  2. border-collapse：collapse 表格中间没有空隙
  3. border-spacing：5px 控制表格 border 之间的距离

```html
<style>
	table {
		width: 600px;
		table-layout:auto;
		border-spacing: 20px;
		border-collapse: collapse;
	}
	td, 
	th {
		border: 1px solid blue;
	}
</style>
```

### 3. img 标签

1. 作用

   1. 发出 get 请求，展示一张图片
2. 属性

   1. alt / height / width / src
   2. src="dog.png" 
   3. alt 图片显示错误时代替的文字  alternative
   4. width="400" 只写宽度，高度会自适应
   5. 高度宽度都写，图片可能会变形，**永远不要让一张图片变形**
3. 事件

   1. onload / onerror 监听图片是否加载成功

   2. 可以在图片加载失败的时候，进行一个挽救，显示一个 404 图片。
4. 响应式

   1. max-wirdth: 100%：为了能在手机上看

```html
<head>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        img {
            max-width: 100%;
        }
    </style>
</head>

<body>
<img id="xxx" width="400" src="dog.png" alt="一只狗子" />
<script>
    xxx.onload = function() {
        console.log("图片加载成功")
    }
    xxx.onerror = function() {
        console.log("图片加载失败")
        xxx.src='/404.png';
    }
</script>
</body>
```

### 4. form 标签

1. input 标签
   1. 作用：发 get / post 请求，然后刷新页面
   2. input 和 button 的区别：input 里面不能再有内容（只能有纯文本），botton 可以还有标签的，也可以加个图片什么的
   3. type：要设置为 submit 才能提交，不写默认为 submit，写错就不能提交。
   4. type=radio 选项，设置相同的 name 属性课设置为单选。
   5. type=checkbox 多选，设置相同的 name 声明多个选项是一组的。
2. 属性
   1. action / autocomplete / method / target
   2. action：就是你要请求的那个页面的地址
   3. action：相当于 src，但这个路径是后台给的，so 不知道，它是点击提交按钮后的执行路径
   4. method：默认为 get  也可以改为 post
   5. autocomplete：自动填充用户信息，给出建议
   6. target=_blank 新开一个页面 和 a 标签里的类似 = 我要提交到哪个页面（会被刷新）
3. 事件
   1. onsubmit：当用户提交的时候触发的事件
   2. type 不是 submit 的话，就不能提交表单  （不写的话 默认也是 submit） 写了别的就会点不动
4. input 的事件
   1. onchange  当用户输入改变的时候触发
   2. onfocus  鼠标移至当前模块时触发
   3. onblur 用户鼠标移开 block
   4. 验证器 required 设置当前 input 为必选项
5. 注意事项
   1. 一般不监听 input 的 click 事件
   2. form 里面的 input 要有 name
   3. form 里要放一个 type=submit 才能触发 submit 事件

![](/images/表单标签.png)

