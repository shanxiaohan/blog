---
layout: post
title: '微信小程序开发配置'
date: 2018-02-03
tags: [小程序]
---

恩这是一篇心酸的回忆录= =

原本是很简单的一个需求：在小程序里推送和健身有关的公众号文章，但是由于平台的限制折腾了好大一圈= =既然微信公众号平台没有开放的接口，那就想爬虫爬下来文章的链接，再在小程序里页面跳转；折腾了一圈微信公众号的爬虫(参考之前的[博客](http://xiaohan77.cn:4000/2018/01/17/2018-01-22-WechatSpider.md/)初步搞了几千条数据准备接着小程序的开发，结果发现了官方文档上的一行字：
> web-view 组件是一个可以用来承载网页的容器，会自动铺满整个小程序页面。`个人类型与海外类型的小程序暂不支持使用。`

(:嗯就酱个人类型的小程序不支持不支持不支持.....emmm不甘心让自己的第一次产品经理经历就此夭折，既然个人类型的不支持，那就看看企业版的能不能申请下来= =首先需要提交公司的营业执照还要盖章验证公司的银行开户账号balabala，于是我这个啥也不懂的小白还上淘宝去搜代办公司营业执照的(傻哭= =)，以为有那个执照号就可以了，没有实体公司也可以，和淘宝客服聊了半天也没太搞懂，晚上给家里打电话聊起这个事儿老爸和我说可以帮我弄一个营业执照(合法的公司)，还帮我跑后续的那些认证手续~(比心老爸) 前前后后提交材料到申请认证完成只花了三天时间，不得不点赞鹅厂客服周六还给我打电话确认信息~ 

当时以为这样就可以开心的写小程序啦~ 直到开始配置域名白名单和业务域名的时候才发现自己真的好天真= =这里的`配置域名白名单`是指小程序要访问的外部接口比如`http`,`https`,`wss`等(其实是不支持`http`请求只支持`https`的)合法域名，而`业务域名`是配置在`<web-view>`中要访问外部html网页时的已备案合法域名。就在配置业务域名的时候突然发现了自己的无知= =以为可以随便加个微信公众号的域名到白名单里，结果还要将小程序后台提供的校验文本放到域名服务器的根目录下= =才明白不是简单加个白名单这么简单，<web-view>允许访问的也只是自己的已备案合法域名，不能访问人家的网页...血崩= =折腾了一大圈还是不能任意访问外网链接...

还是不死心，又心生一计= =既然文章的URL都已经爬下了，那就把内容都下到自己的服务器上，就可以在<web-view>里配置自己服务器的合法域名了~ (好在之前有个腾讯云的服务器备案过域名= =)

接下来就要根据文章的URL把html页面保存到服务器上....emmmm现在来把要做的工作整理一下思路好了：
![整体架构图](/images/blog/project-flow.png)

恩有木有清晰了一点= =现在来一样一样细说~  
##### 0. 域名备案
略= =  
##### 1. nginx配置
先来配置一下nginx下的HTTPS吧~ 小程序管理后台和腾讯云关联授权后给分配个CA证书，包含`.crt`和`.key`文件，上传到nginx服务器的根路径下，在`conf.d/`路径下添加自己的规则，配置HTTPS和请求静态资源的接口，参考代码如下：
```shell
server {
  listen 443;
  server_name xiaohan77.cn; #填写绑定证书的域名
  ssl on;
  ssl_certificate conf.d/1_xiaohan77.cn_bundle.crt;
  ssl_certificate_key conf.d/2_xiaohan77.cn.key;
  ssl_session_timeout 5m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#按照这个套件配
置
  ssl_prefer_server_ciphers on;

  location / {
    root  /usr/share/nginx/html;
    index  index.html index.htm;
  }

  #请求图片资源
  location ~* /img/.*/.*\.(jpg|gif|png|jpeg)$ {
    gzip on;
    gzip_http_version 1.1;
    gzip_comp_level 2;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    root /home/ubuntu/project/spider/static;
  }

  #请求html页面
  location ~* /article/.*\.html$ {
    add_header Access-Control-Allow-Origin *;
    root /home/ubuntu/project/spider/static;
  }
}
```
##### 2. 模拟请求 && 解析数据
根据文章的URL将html页面和图片下载到本地，并将图片的链接改成本地路径，部分代码如下（参考注释）：

```js
var processData = async (url, fileid) => {
  request(url, function (err, response, body) {
    if (!err && response.statusCode == 200) {
      let content = body.toString();
      // 正则匹配script标签，原脚本可能对img标签的链接改写
      let reg1 = /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi;
      content = content.replace(reg1, '');
      content = content.replace('\n\n', '');

      // 正则匹配所有的图片URL，后面下载到本地
      let reg2 = /<img(.*?) src="http:\/\/mmbiz.qpic.cn\/mmbiz\/(.*?)"/gi;
      let res2 = content.match(reg2);
      let idx = -1;
      //asnyc库提供异步+循环保证次序的方法
      //item参数为遍历res2数组的每个元素，callback为map的第二个参数回调函数
      async.map(res2, (item, callback) => {
        //正则匹配URL中的src路径
        let src = /src=\"(.*?)\"/.exec(item);
        let match_url = src[1];
        let type = match_url.split('wx_fmt=')[1] || 'jpg';
        content = (() => {
          idx += 1;
          let article_path = "/home/ubuntu/project/spider/static/img/" + fileid;
          if (!fs.existsSync(article_path)) {
            fs.mkdirSync(article_path);
          }
          let img_src = article_path + '/' + idx + "." + type;
          //download库异步请求图片资源
          download(match_url).then(data => {
            fs.writeFile(img_src, data, function (err) {
              if (err) {
                console.log(err, "download fail");
              }
            });
          });
          let new_path = "../img/" + fileid + '/' + idx + "." + type;
          //将图片的src链接替换为本地路径
          content = content.replace(match_url, new_path);
          return content;
        })();   //闭包立即执行
        callback(null, content);
      }, (err, res) => {
        // img的全部路径替换完成将整个HTML文件存到本地
        // 【注意】上面的map循环时，content每次更新需要用匿名函数包起来立即执行，否则在异步调用的时候会先执行这里的回调函数，导致没有执行完异步结果就将content写入文件导致错误
        fs.writeFile('/home/ubuntu/project/spider/static/article/' + fileid + '.html', content, function (err) {
          console.log(err);
        });
      });
    }
  });
}
```
上面代码中需要注意的是循环＋异步执行的部分，网上也有很多解决方案，如果没有处理则会优先顺序执行循环和异步后面的部分，再在回调中处理结果，但是这样不符合我们的预期：循环结束后将更新结果保存到本地（参考上面的注释）；在循环内部也要加个匿名函数的闭包立即执行才，否则循环内的变量取值都是循环结束的变量。恩这只是其中一种解决方案，应该还有更优雅的写法= =

嗯数据和接口都差不多啦~ 可以接着回来开发小程序的第三个页面啦~\(≧▽≦)/~