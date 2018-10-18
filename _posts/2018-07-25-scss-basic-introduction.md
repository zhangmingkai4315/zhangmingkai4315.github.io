---
id: 81
title: SCSS基本用法笔记
date: 2018-07-25T17:15:44+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=81
permalink: /2018/07/scss-basic-introduction/
categories:
  - frontend
tags:
  - web
  - 前端
  - 开发技术
format: aside
---
#### 基础用法

##### 1&#46; 变量

注意引用的时候使用中划线-和下划线_连接的定义是相同的比如下面

    $link-color: blue;
    a { color: $link_color; }
    

##### 2&#46; 变量引用

变量引用可以用在任何可以替换的位置

    $highlight-color: #F90;
    $highlight-border: 1px solid $highlight-color;
    .selected { 
       border: $highlight-border; 
     }
    

##### 3&#46; 嵌套规则

嵌套规则有两类 一类是使用空格替换的嵌套 比如我们定义如下的内容中：

    #content {
     article { h1 { color: #333 } 
     p { 
        margin-bottom: 1.4em } 
     } 
     aside { 
        background-color: #EEE 
     } 
    }
    

编译完成后变为：`#content artical {...}` 等css父类选择器，而对于一些伪类的时候比如`#content:hove{...}`的选择则需要使用另外一种选择器

    #content {
      &:hover{
    
      }
    }
    

如果需要为content中的所有h1,h2,h3元素设置一个固定margin的话可以使用下面的群组嵌套：

<pre><code class="scss">#content {
  h1,h2,h3{
      margin:10px;
  }
}
</code></pre>

除了父级选择器的使用以外，还可以调用其他类别的选择器比如直接子元素选择>，相邻选择+和同层选择~。如下面所示：

<pre><code class="scss">article {
  ~ article { border-top: 1px dashed #ccc }
  &gt; section { background: #eee }
  dl &gt; {
    dt { color: #333 }
    dd { color: #555 }
  }
  nav + & { margin-top: 0 }
}
</code></pre>

转化后的css代码如下：

<pre><code class="css">article ~ article { 
   border-top: 1px dashed #ccc 
} 
article &gt; footer { 
    background: #eee 
} 
article dl &gt; dt {
    color: #333 
} 
article dl &gt; dd { 
    color: #555
} 
nav + article { 
    margin-top: 0 
}
</code></pre>

其中最后一条nav+article是一个前向的叠加使用&来组合

对于嵌套的使用还可以借助与属性嵌套来完成css构建比如下面的SCSS代码中通过属性嵌套来定义border-left和border-right.

    nav { 
      border: 1px solid #ccc { 
        left: 0px; right: 0px; 
     } 
    }
    

转换后的代码如下：

    nav { 
        border: 1px solid #ccc; 
        border-left: 0px; 
        border-right: 0px; 
    }
    

##### 4&#46; 导入scss模块

原生的css导入import仅仅在执行到该语句的时候，发送请求下载额外的css或者其他资源，而借助与scss的@import可以同时用于模块的管理方式和快速的加载速度（所有文件最终存储在同一个 css文件).

假如需要导入的文件仅仅是定义变量的文件，可以使用_来定义文件名，在导入的时候去掉下划线即可，比如文件`theme/_dark.scss`中定义了黑色背景的一系列变量，我们在多个文件中引入的时候可以使用@import &#8220;theme/dark&#8221; 以下的三种情况下会使用css的原生import: &#8211; 被导入文件的名字以.css结尾；

  * 被导入文件的名字是一个URL地址（比如http://www.sass.hk/css/css.css）

  * 被导入文件的名字是CSS的url()值。

##### 5&#46; 设置变量的默认值

任何之前存在的定义或者引入的文件中存在定义都会覆盖缺省的值，缺省的定义如下：

    $fancybox-width: 400px !default; 
    .fancybox { 
        width: $fancybox-width; 
    }
    

##### 6&#46; 注释文字的显示

使用注释文字的话方式不同会导致注释在文件中的显示不同，

    body { 
        color: #333; // 这种注释内容不会出现在生成的css文件中 
        padding: 0; /* 这种注释内容会出现在生成的css文件中 */ 
    }
    

##### 7&#46; 混合器（内联）

如果我们有一些代码块在scss代码中被重复的使用可以借助与混合器的方式来编写代码结构比如做一个边角的css样式中可能整个网站很多元素都会用到，这个时候使用下面的代码定义代码块：

<pre><code class="scss">@mixin rounded-corners { 
  -moz-border-radius: 5px; 
  -webkit-border-radius: 5px; 
  border-radius: 5px; 
}
</code></pre>

我们的其他代码中可以直接使用该代码，仅仅需要使用@include即可

    notice { 
        background-color: green; 
        border: 2px solid #00aa00; 
        @include rounded-corners; 
    }
    

另外scss也可以定义混合器传递参数比如我们定义的代码块有时候需要一些固定的样式，

    @mixin link-colors($normal, $hover, $visited)
    { 
    color: $normal; 
    &:hover { color: $hover; } 
    &:visited { color: $visited; } 
    }
    

其他的地方使用时候可以：`a { @include link-colors(blue, red, green); }`

如果传递的参数类似如下的代码，则可以只传递一个参数其他的使用默认的normal值既可以。

    @mixin link-colors(
        $normal,
        $hover: $normal,
        $visited: $normal
      )
    {
      color: $normal;
      &:hover { color: $hover; }
      &:visited { color: $visited; }
    }
    

##### 7&#46; 继承的使用

scss中定义了继承的用法，可以通过@extend来完全的继承另外一个变量的值，甚至是该值的其他的属性

    .disabled {
       color: gray; 
       @extend a; 
    }
    

@extend背后最基本的想法是，如果`.seriousError {@extend .error}`， 那么样式表中的任何一处.error都用.error.seriousError这一选择器组进行替换。这就意味着相关样式会如预期那样应用到.error和.seriousError。

尽管混合器可能导致代码量增加，但混合器可以执行参数传递，继承本身是不可以。假如创建的混合器没有传参数，那您可能就要考虑下是否选择继承会更合适。

#### 使用SASS的问题

    .button { 
       display: block; 
       padding: 10px; 
       background: green; 
    } 
    .sidebar .signup .button { 
        margin-top: 20px; 
    } 
    .registrantion, .remember-password { 
       .button { margin-bottom: 33px; }
     } 
    .edit-account .delete-area .button { 
        background-color: red; 
        color: white; 
    } 
    .article a { @extend .button; }
    

使用继承本来仅仅需要扩展article a的样式，但是在编译中所有存在.button的地方都被重复的加入了新的article a的代码。

SASS中placeholder可以解决继承带来的代码重复的问题。定义如下：

    .button, 
    %button { 
      display: block; 
      padding: 10px; 
      background: green; 
    } 
    .sidebar .signup .button { 
      margin-top: 20px;
    } 
    .registrantion, .remember-password { 
      .button { 
         margin-bottom: 33px; 
       }
      } 
    .edit-account .delete-area .button { 
       background-color: red; color: white; 
    }
    .article a { @extend %button; }