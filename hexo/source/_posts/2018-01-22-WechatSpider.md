---
layout: post
title: '微信公众号文章爬虫实战'
date: 2018-01-22
tags: [微信公众号爬虫]
---

#### 前序
为了开发之前[博客]((http://xiaohan77.cn:4000/2018/01/17/2018-01-17-%E5%B0%8F%E7%A8%8B%E5%BA%8F%E6%97%A5%E5%8E%86%E7%BB%84%E4%BB%B6%E5%BC%80%E5%8F%91/) 中提到的小程序的第三个选项卡功能:每日推送和健身有关的公众号文章（如：硬派健身、keep等），由于新版小程序官方开放了`<webview>`组件嵌入外部链接的能力，需要指定组件的`src`即可打开链接内容，因此需要准备这些公众号文章的链接，再依次填充到组件里即可。然而爬取微信公众号文章的过程并没有那么顺利... 于是有了这篇踩坑经历= =

#### 一、爬虫初试之"搜狗微信"
偶然间看到[搜狗微信](http://weixin.sogou.com/)这个网站可以搜公众号的文章，心想写个爬虫直接爬这个网站里面的历史文章链接不就好了嘛~ 于是在服务器上搭个scrapy准备开爬= = 开爬之前顺手点开github搜搜有没有类似的小项目参考一下~ 一搜就搜到了这个[WechatSogou](https://github.com/Chyroc/WechatSogou)哇一看1600多个赞应该很靠谱~ 顺手装了一下试了试，我只需要得到文章列表的url信息，直接几行代码搞定：

```python
ws_api = wechatsogou.WechatSogouAPI()
res = ws_api.get_gzh_article_by_history('硬派健身')
article = res['article']
for item in article:
    print(item)
```
emmmm当然事情不会这么简单= =下午试的时候还好，晚上回来又多测了几次就gg了... 提示要输入图片验证码，然而我的腾讯云主机没有图形界面不能弹出验证码图片，看issues里有说连到打码平台上提交的= =但是后来又发现通过搜狗微信的接口返回的公众号文章链接是临时链接，过了有效期即失效（嗯第二天早上再打开昨晚爬的链接就已经失效了/(ㄒoㄒ)/~~）于是放弃了搜狗微信的这条路= =

#### 二、爬虫初试之"Charles抓包"
之前写网页端爬虫的时候用过这样的方法：先用抓包工具（如windows下的fiddler或Mac下的Charles）分析客户端发起的真实请求，再模拟其`request`的`query`和`headers`，当然还有很重要的`cookie`参数，测试出一定的规律再自己模拟请求得到返回结果（可能是加密过的数据）。
顺着上面的思路，先来看看在微信客户端点击`历史文章`是如何发起请求得到数据的~ 据说fiddler也非常好用，不过我一直用自己的Mac，就还用Charles顺手一点~ （之前下的版本太低已经打不开了= =）下面一步一步从安装抓包工具`Charles`开始= =

##### 安装Charles4.0.1破解版
参考[这篇博客](http://www.sdifen.com/charles401.html)~ 破解很简单，替换个jar包即可~ 但是期间出现了一点点的小问题：替换了jar包之后再打开会提示`Charles.app资源已损坏`，这是由于升级了新版的MacOS Sierra之后默认关闭了`允许从任何来源下载`，而且在设置里面是不显示该选项的，可以在命令行里输入以下代码显示该选项，勾选上再打开即可~
```shell
    sudo spctl --master-disable     #注意master前面是两个连字符
```

##### 配置Charles抓包iphone客户端(https)请求
首先参考这篇[博客](https://www.jianshu.com/p/595e8b556a60)~过程大概类似这里就不过多重复了~ 
还有一个需要配置的地方是，发现返回的数据是乱码的，参考[这篇博客](http://blog.csdn.net/a327369238/article/details/52856833
)，重点是第二步设置`proxy`里面的`SSL Proxy Settings`为`*:443`

##### 模拟请求获取历史消息列表
接下来就和普通的网页爬虫一样啦~ 从Charles拷贝出请求的headers和cookie，加上python的requests库，对响应结果正则处理~ 最后我们想要的数据是写在js里面的`msgList`变量，数据处理部分的代码如下：

```python
html = HTMLParser.HTMLParser()
rex = "msgList = '({.*?})'"
pattern = re.compile(pattern=rex, flags=re.S)
match = pattern.search(html_content)
if match:
    data = match.group(1).decode("utf-8")
    data = html.unescape(data)
    data = json.loads(data)
    articles = data.get("list")
    return articles
```
这样就可以获取到历史文章的title(文章标题), content_url(文章链接), source_url(原文链接), digest(摘要), cover(封面图)等信息~ emmmm但是尼= =这个cookie和header中的一些参数(比如 X-WECHAT-KEY和每个公众号的验证有关)不是所有公众号都是通用的，而且cookie的有效期只有几个小时，折腾半天也只能拿到最新的10条数据，如果要再抓取其他公众号或者更早的文章这样的效率就很低= =不过聊胜于无吧~ 最起码现在数据是有了，把文章链接等数据先存到数据库里~

#### 三、终极大招之AnyProxy
偶然在知乎上发现了这个神器`AnyProxy`，是一个http代理工具，并且利用中间人攻击的原理可以对`https`请求进行劫持和伪造修改（需要配置信任CA证书），[参考文档](http://anyproxy.io/cn/)也写的很明白。AnyProxy的神奇之处在于可以通过js脚本配置拦截请求的处理请求逻辑，实现以下的功能：

- 拦截并修改正在发送的请求：
    可修改内容包括请求头（request header)，请求体（request body），甚至是请求的目标地址等
- 拦截并修改服务端响应：
    可修改的内容包括http状态码(status code)、响应头（response header）、响应内容等
- 拦截https请求，对内容做修改
    本质是中间人攻击（man-in-the-middle attack），需要客户端提前信任AnyProxy生成的CA

对应自己项目的需求，整理一下爬虫的大致思路如下图：（手残党吐血献上= =）
![spider-flow](/images/blog/spider-flow.png)

对应这个爬虫的需求，我们可以拿到请求头的校验参数以及cookie、直接得到返回结果、甚至还可以对返回的页面注入js代码实现页面跳转（亲测注入JS代码可以跳转到其他页面）嗯试用了一下发现真的很赞！只需要在启动AnyProxy的时候指定rule脚本，就像这样`anyproxy -i --ignore-unauthorized-ssl --rule rule.js`, 编写rule规则可以参考官方文档，以下是官方的demo，在收到服务器返回的真实response之后修改body的内容：
```js
module.exports = {
  *beforeSendResponse(requestDetail, responseDetail) {
    if (requestDetail.url === 'http://httpbin.org/user-agent') {
      const newResponse = responseDetail.response;
      newResponse.body += '-- AnyProxy Hacked! --';
      return new Promise((resolve, reject) => {
        setTimeout(() => { // delay the response for 5s
          resolve({ response: newResponse });
        }, 2000);
      });
    }
  },
}
```

根据这个demo重写了自己的规则(参考一下代码注释)，将得到的真实相应结果返回自己的本地服务器，再将数据本地化：
```js
*beforeSendResponse(requestDetail, responseDetail) {
    // 匹配历史文章页面URL
    if (/mp\/profile_ext\?action=home/i.test(requestDetail.url)) {
        //解析参数拼接成对象发送给本地服务器
        let reqHeaders = requestDetail.requestOptions.headers,
            resCookie = responseDetail.response.header['Set-Cookie'],
            setcookie = '';
        for (let item of resCookie) {
            setcookie += item.split('; ')[0] + '&';
        }
        setcookie = setcookie.substring(0, setcookie.length-1);
        var params = {
            setcookie,
            reqcookie: reqHeaders['Cookie'],
            wechatkey: reqHeaders['X-WECHAT-KEY'],
            wechatuin: reqHeaders['X-WECHAT-UIN']
        };
        // 文章的标题、链接等信息保存在msgList参数中
        var reg1 = /var msgList = \'(.*?)\';/,
            reg2 = /window.appmsg_token = \"(.*?)\";/;
        if(responseDetail.response.body) {
            let resp_data = responseDetail.response.body.toString();
            var ret = reg1.exec(resp_data);
            var appmsg_token = reg2.exec(resp_data)[1];
            params['appmsg_token'] = appmsg_token;
            //将提取到的数据发送给本地的后台服务器
            HttpPost(ret[1],requestDetail.url, params, "/fetchHistList"); 
        }else {
            console.log("*********Error response**********");
            console.log(responseDetail.response);
        }
    }
  }
```
上面用到的HttpPost方法如下，将提取到的数据发送给本地的后台服务器:

```js
//将json发送到服务器，str为json内容，url为历史消息页面地址，path是接收程序的路径和文件名
function HttpPost(str,url,params,path) {
    var data = {
        content: str,
        url: encodeURIComponent(url),
        params: encodeURIComponent(querystring.stringify(params))
    };
    content = querystring.stringify(data);
    var options = {
        method: "POST",
        host: "xiaohan77.cn",
        port: 3001,
        path: path,     //接收程序的路径和文件名
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
            "Content-Length": content.length
        }
    };
    var req = http.request(options, function (res) {
        res.setEncoding('utf8');
        res.on('data', function (chunk) {
            console.log('BODY: ' + chunk);
        });
    });
    req.on('error', function (e) {
        console.log('problem with request: ' + e.message);
    });
    req.write(content);
    req.end();
}
```
以上就是利用AnyProxy截获公众号历史文章页面并发送给本地服务器的部分，对应的还需要有后台的接口接收post过来的数据，这个本地的后台服务器我用了一个nodejs的MVC框架koa2，对应的`/fetchHistList`接口部分代码如下：
```js
const qs = require('querystring');
const URL = require('url');
const unescape = require('unescape-html');  //用于反转义html标签
const request = require("request");
'POST /fetchHistList': async(ctx, next) => {
    let url = decodeURIComponent(ctx.request.body.url),
      content = ctx.request.body.content,
      params = ctx.request.body.params;
    let params_obj = qs.parse(qs.unescape(params));
    let {
      setcookie,
      reqcookie,
      wechatkey,
      wechatuin,
      appmsg_token
    } = params_obj;
    let parse_url = URL.parse(url).query.split('&'),
      biz = parse_url[1].slice(6);
    let time = Math.round(Date.now() / 1000); //转换为10位时间戳
    // Account对应数据库的和公众号信息有关的一个单独的表
    let account_data = await Account.findOne({
      where: {
        biz: biz
      }
    });

    if (!account_data) {
      await Account.create({
        name: '',
        biz,
        time,
        setcookie,
        reqcookie,
        wechatkey,
        wechatuin,
        appmsg_token
      });
    } else {
      // 已经存在该公众号信息，更新cookie等参数
      await Account.update({
        time,
        setcookie,
        reqcookie,
        wechatkey,
        wechatuin,
        appmsg_token
      }, {
        where: {
          biz: biz
        }
      });
    }
    // parseHistList(content);
    //构造新的请求获取更多历史文章列表
    await fetchMoreMsg(biz, url, params_obj, 0);
    
    ctx.response.body = {
      code: 200,
      msg: 'success'
    };
  }
```
上面代码的部分主要是将一些校验参数保存起来，为了构造请求获取更多的历史文章，到这里先来整理一下思路：上面得到历史文章的页面是需要手动在手机客户端点击公众号的`查看历史文章`按钮（或者使用模拟器工具），通过AnyProxy截获对微信服务器的请求响应，处理后发送给本地的后台服务器，这样存在的问题是，如果想要获取更多的数据就需要手动向下滑动加载更多，这样就不能很好的自动化= =后来意外发现在请求更多数据的时候返回的是干净的json数据，可以通过`offset`和`count`连续请求历史数据，这样其实可以不用提取解析主页面返回的十条数据，根据已经截获到的header和cookie可以轻松地伪造`获取更多`的请求，主要实现部分如下：
```js
const fetchMoreMsg = async(biz, reqUrl, params, offset) => {
  let { setcookie, reqcookie, wechatkey, wechatuin, appmsg_token } = params;
  let setcookies = qs.parse(setcookie);
  let is_continue = 0,
    pass_ticket = setcookies['pass_ticket'],
    wap_sid2 = setcookies['wap_sid2'];
  let getMsgUrl = 'https://mp.weixin.qq.com/mp/profile_ext?action=getmsg&__biz=' + biz + '&f=json&offset=' + offset + '&count=10&is_ok=1&scene=124&uin=777&key=777&pass_ticket=' + pass_ticket + '&wxtoken=&appmsg_token=' + appmsg_token + '&x5=0&f=json';
  let headers = {}, wxtokenkey = '';
  for (let item of reqcookie.split('; ')) {
    if (item.split('=')[0] === 'wxtokenkey') {
      wxtokenkey = item.split('=')[1];
    }
  }
  let cookie = 'devicetype=iOS10.2.1; lang=zh_CN; pass_ticket=' + pass_ticket + '; version=16060125; wap_sid2=' + wap_sid2 + '; wxuin=1190142921; rewardsn=; wxtokenkey=' + wxtokenkey;
  headers = {
    'Host': 'mp.weixin.qq.com',
    'Cookie': cookie,
    'Connection': 'keep-alive',
    'User-Agent': 'Mozilla/5.0 (iPhone; CPU iPhone OS 10_2_1 like Mac OS X) AppleWebKit/602.4.6 (KHTML, like Gecko) Mobile/14D27 MicroMessenger/6.6.1 NetType/WIFI Language/zh_CN',
    'Referer': reqUrl,
    'Accept-Language': 'zh-cn',
    'X-Requested-With': 'XMLHttpRequest',
    'Accept': '*/*'
  };
  let options = {
    method: 'GET',
    url: getMsgUrl,
    headers: headers,
  };

  request(options, function (error, response, body) {
    if (error) throw new Error(error);
    body = JSON.parse(body);
    if (body['ret'] === 0) {
      is_continue = body['can_msg_continue'];
      offset = body['next_offset'];
      let msg_list = JSON.parse(body['general_msg_list']).list;
      if (msg_list && msg_list.length > 0) {
        try {
          (async () => {
            await saveArticle(biz, msg_list);   //将msg_list数据保存到MySQL，这里不再详说
            if (offset > 0 && is_continue > 0) {
              setTimeout(() => {
                (async () => {
                  await fetchMoreMsg(biz, reqUrl, params, offset);
                })();   //闭包立即执行，否则offset不立即生效
              }, 1000);
            }
          })();
        } catch (error) {
          console.log(error, '!!!!!!error!!!!!!');
        }
      }
    }
  });
}
```
至此就大功告成了一半啦~ 循环不断更新`offset`并通过返回的`can_msg_continue`参数判断是否可以继续请求~ 嗯上面的代码只是为了实现功能做的demo测试，写的并不是很优雅= =比如循环+异步请求的部分(网络IO、数据库操作还有延时setTimeout)这些事件驱动的回调机制都可能带来问题，后面还会接着详细说这个问题~~