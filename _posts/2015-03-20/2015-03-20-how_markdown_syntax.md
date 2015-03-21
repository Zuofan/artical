---

layout: post
title: Kramdown 使用
category:  工具使用
tags: [Kramdown]

---

本文主要介绍了`Kramdown`方面的一些语法。

### 修饰字词
1. `**强调**`	：	**强调**
2. `*斜体*`		： 	*斜体*
3. 水平线 `- - - -` ： 
4. `{::comment} content ignored {:/comment}`
	下面的文字不会出现在HTML文档中：
	{::comment}
	This is a comment!
	{:/comment}
5. `[link_name](link_address)`:
	[能止]({{ site.BASE_PATH }})

    `![Images](/assets/ico/logo.png)`，如：
	![Images]({{ site.BASE_PATH }}/assets/ico/logo.png)
4. `Blockquotes` :

- `>`	表示单层的blockquote
- `>>`	表示嵌套的blockquote

>A sample blockquote.

>>Nested blockquotes are also possible.
>>上面的空行是必要的，否则，嵌套的`blockquotes`是没有作用的。

> ## Headers work too
> This is the outer quote again.

### 列表
* 列表方式(`*`)
- 列表方式(`-`)
+ 列表方式(`+`)

1. Sequence(`1. `)
2. Sequence(`2. `)

### 代码
- 代码块

		def what?
			42
		end	

   下面这种方式使用新的`kramdown`语法来书写，但高亮所需的条件在**github**中没有，而且缩进的方式也比较麻烦，故而需要采用其他的方式才可以。


		~~~ javascript
		function add(a, b) {
			return a+b;
		}
		~~~

- 代码高亮
  在`kramdown`语法中的语法高亮不是很方便，需要安装`gem install coderay`才可以使用，而且，在github上没有coderay，所以最好使用自带的语法高亮进行显示。在这一点上，没有`vimwiki`方便。

  **采用外接的代码高亮方式：**
  1. 下载[jQuery Syntax Highlighter - Based on Google's Prettify](http://balupton.github.io/jquery-syntaxhighlighter/demo/)
  2. `jquery-syntaxhighlighter`设置
	首先配置**jquery.min.js**，因为*jquery-syntaxhighlighter*需要**jquery**的支持
		
		<script type="text/javascript" src="http://ajax.googleapis.com/ajax/libs/jquery/1.4.2/jquery.min.js"></script>
	
	其次，需要配置**jquery-syntaxhighlighter**

		<!-- Include jQuery Syntax Highlighter -->
		<script type="text/javascript" src="http://balupton.github.com/jquery-syntaxhighlighter/scripts/jquery.syntaxhighlighter.min.js"></script>

	最后，设置**jquery-syntaxhighlighter**的启动参数

		$("pre").addClass("highlight");
		$.SyntaxHighlighter.init({
			//如果要引用本地的资源，需要打开下面的注释。默认使用在线的资源，具体可参jquery.syntaxhighlighter.js
			//'prettifyBaseUrl': '/assets/resources/jquery-syntaxhighlighter/prettify',
			//'baseUrl': '/assets/resources/jquery-syntaxhighlighter'
		});


### Math block

目前直接在这里写好像不支持（`待研究`）
