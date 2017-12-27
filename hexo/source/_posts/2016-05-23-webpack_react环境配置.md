---
layout: post
title: 'SublimeText3下webpack+React环境配置'
date: 2016-05-23
tags: [webpack]
---

#### 一、SubimeText3配置
1、需要安装的插件（package）

快捷键command + shift + p 打开install package入口，输入需要安装的插件名称

下面是几个常用的package： 

1. Emmet 代码自动补全插件

	输入div#main后按Tab键，生成如下代码：[或元素.id名称]
	
	<div class="main"></div>
	
2. CSS3
3. SublimeLinter和SublimeLinter-contrib-standard 代码格式化校验（错误提示）
	注意SublimeLinter要安装for Sublime Text 3版本的
	
	安装后在Tools下拉菜单里出现SublimeLinter的选项，首先点击toggle linter，默认安装后为enabled，开启校验，也可以关闭；
	
	选择show errors on save,设置在保存文件时候校验错误；

4. AdvancedNewFile 在项目下任何位置新建文件/文件夹

	快捷键打开: alt + command + n 输入新建文件相对路径
	
5. Babel 可编译ES6，JSX代码，为其语法高亮，安装完设置将JS代码用JavaScript(Babel)编译

#### 二、webpack配置

1、demo项目目录结构  
说明：  
1）首先新建工程startup，将附件中的webpack和.babelrc, package.json文件拷到项目目录下.
.babelrc运行时自动加载的配置文件用于说明babel可以预设的转码格式;

2）package.json文件配置一些需要的加载器，依赖关系，环境变量还有测试相关的参数；

3）webpack文件夹下有三个文件：

path.js用于定义一些路径相关的常量；
webpack.config.dev.babel.js用于设置项目打包的入口出口路径，react-hot-loader热加载等，其中entry的path.join(paths.src, 'index.js')为项目的入口文件，
output配置项目的出口文件，其中path: paths.dist定义经过加载打包后的输出目录（自动生成）
filename: 'js/[name].js'指定将src源码中的js文件以原文件名.js的形式放在dist/js/下.

webpack.config.prod.babel.js和webpack.config.dev.babel.js类似，区别在于.dev.是用于开发环境下的配置信息，.prod.用于上线环境的配置信息

2、运行项目
首先npm i 安装配置文件所需的node_modules
1）开发：
在命令行中项目目录下输入npm run dev，开启本地3000端口，可以在浏览器打开看到；

2）上线：
在命令行中项目目录下输入npm run build，完成编译打包

3）demo
上述目录结构下的demo文件夹用于写自己的demo展示（HTML文件），通常开发react组件时在src目录下写js文件，可以将用于展示的HTML文件放在demo文件夹下，不影响组件的开发以及后面的上线打包，这样需要在path.js下增加一个常量const demo = `${base}/demo`，在webpack.config.dev.babel.js文件修改contentBase: paths.demo

3、import外界的CSS模块
如在sub.css文件中定义 .wrap {background: #f0f;}
可以在index.js中引入，import Styles from './sub.css'
使用该对象：
<div style={Styles.wrap}></div>
