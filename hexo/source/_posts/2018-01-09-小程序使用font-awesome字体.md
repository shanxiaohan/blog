---
layout: post
title: '小程序使用font-awesome字体'
date: 2018-01-09
tags: [小程序]
---

#### 微信小程序引用font-awesome字体

1、参考文档
[小程序引用font-awesome字体](http://blog.csdn.net/yiyingcsdn/article/details/71215854)

2、在component自定义组件中使用遇到的问题
按照上面文档的步骤将转换后的CSS拷贝到`app.wxss`文件中，在`pages`下的页面是可以正常引用全局的样式属性的，但是在自定义组件内应用的样式有很多限制，要仔细看[开发文档](https://mp.weixin.qq.com/debug/wxadoc/dev/framework/custom-component/wxml-wxss.html)的要求，这里列举出来特殊注意一下：

- 组件和引用组件的页面不能使用id选择器（#a）、属性选择器（[a]）和标签名选择器，请改用class选择器；
- 组件和引用组件的页面中使用后代选择器（.a .b）在一些极端情况下会有非预期的表现，如遇，请避免使用；
- 子元素选择器（.a>.b）只能用于 view 组件与其子节点之间，用于其他组件可能导致非预期的情况；
- 继承样式，如 font 、 color ，会从组件外继承到组件内；
- 除继承样式外， app.wxss 中的样式、组件所在页面的的样式对自定义组件无效；

上面的最后一句话就表明了在`app.wxss`添加的关于font-awesome字体的样式在自定义组件中是不能生效的，并且也不能用`@import`引用外部的属性，因此将前面转换的CSS代码拷贝到组件内部的wxss文件中= =但是问题又来了...在`app.wxss`可以生效的代码，拷贝到组件里面的样式文件会报错= =原来是冒犯了上面注意事项的第三条，里面的一些class用到了子元素选择器，因此，只将font-awesome.css文件里面自己需要的class样式单独挑出来，确保没有其他的选择器导致报错= =
