---
title: chrome network canceled 问题排查记录
date: 2017-06-07 22:27:19
tags: chrome 302 canceled
---

今天项目组反应他们的应用切系统在做跳转的时候，会停留在一个空白页。
介绍一下背景，项目里采用的cas做单点登录，切系统（从a系统通过超链接进入b系统）的时候会经过几次302.

**原因初探**

我本机打开目标系统按照步骤果然重现问题，打开chrome的开发者工具，抓包下来发现浏览器进入空白页之前的最后一个请求是canceled状态的。

![chrome canceled network][1]


看到这个问题时有点烦，因为之前解决过这个问题，怎么又来。

之前有一段js，也是cas用的，登录时如果service ticket生成成功，进行302跳转到目标系统，当时遇见过这个问题。当时记得google了几个答案，一般说时html dom没有渲染完成或一个网络请求没有结束就进行302跳转会出这种问题，当时的改进方案时在ajax的complete周期进行跳转。当时那个问题也是偶发，改完后观察一段时间也没在遇见过，也就以为解决了（现在看来那个可能还真不是问题原因）。

回到今天的问题，今天的场景和之前遇见过的还不一样，因为这次跳转没有经过js代码，是服务器端response回来的302，也就是排除了之前那种可能。

仔细研究下这个canceled的请求报文：
![请求报文详情][2]


可以看到，没有任何游泳的线索。

**柳暗花明**

跳出之前的思路，根据项目组的反馈，之后跳到某一个特定的系统是偶尔会出现这种情况，跳到其他系统没这问题。沿着这条线往下查，首先想到对比进入两个系统的http报文。显然使用fiddler抓包并保存response报文为文本然后通过文本对比两个报文更容易发现不同的地方。

![fiddler 保存完整response为文本][3]

启用fiddler之后还有一个新的发现，之前在chrome里看到的那个canceled的那个请求，其实是有发到服务器的，而且服务器也有响应。

然后对比了两个系统的response回302的两个网络请求的响应报文，发现一个不一样的地方。
302的response是告诉浏览器跳转到另外一个地址上去，具体的就是通过http response header中的location字段，这个字段里方的是一个url，对比发现两个系统返回的url格式有点区别：(下面非真实数据)

正常系统：Location: http://www.abc.com/;JESSIONID=ASFASDFASDFASDFASDF
问题系统：Location: http://www.def.com;JESSIONID=DDFSDFSDFSDFSDFSDFSDF

区别就在没问题的系统地址后面有一个**斜杠**。

这两个url后面都跟着JSESSIONID， 这个是一个携带cookie的手段，在浏览器禁用cookie的时候，通过url直接携带cookie。

**问题解决**

两个报文对比完成之后，通知项目组所有的项目地址配置后面加一个斜杠就好了，问题也解决了。

**跟进**

想不通为什么浏览器在这里矫情一个斜杠呢？

首先自己写了一个简单的服务器程序确认斜杠问题

```go
func HelloServer(w http.ResponseWriter, r *http.Request) {
	//chrome will redirect to the target url
	//	http.Redirect(w, r, "http://test.hongkai.cn:85/;jsessionid=3371D97462347B30C7D55ABD9C455E31", http.StatusFound)
	//chrome will canceled the request because the url is not valid
	http.Redirect(w, r, "http://test.hongkai.cn:85;jsessionid=3371D97462347B30C7D55ABD9C455E31", http.StatusFound)
}

func MainServer(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "hello, world!\n")
}

func main() {
	http.HandleFunc("/hello", HelloServer)
	http.HandleFunc("/", MainServer)
	err := http.ListenAndServe(":85", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
```
完整代码：https://github.com/HongkaiWen/gostudy/blob/master/httpserver/server.go

刚开始测试时，我没有带上jsessionid，并没有重现问题，后来加上jsessionid问题重现：

| 测试批次    |  预期正确地址   |  预期错误地址   | 
| --- | --- | --- |
|  1   |  http://test.hongkai.cn:85/   | http://test.hongkai.cn:85    |
|  2   | http://test.hongkai.cn:85/;jsessionid=...   |  http://test.hongkai.cn:85;jsessionid=...   | 

*（这个域名是我在hosts文件里配到本机的）*

第批次的测试，全部都是ok的，chrome并没有canceled的网络请求，第二批次的测试重现了问题，所以问题应该就是第二批次的测试case上：
http://test.hongkai.cn:85;jsessionid=...
所以怀疑这个url是不是一个非法的url呢？
通过浏览器直接访问地址：
http://www.baidu.com;jsessionid=abc
浏览器直接给跳google去了，说明浏览器认为这个url不是一个合法的url

此时问题原因清晰了，浏览器认为这种地址
http://www.baidu.com;jsessionid=abc
不是一个合法的url地址，浏览器并没有把他当成url处理。
**在服务器端返回302响应的时候，告诉浏览器跳到一个他认为不合法的地址上，浏览器取消了这个请求。**


再次证明一个道理：所有诡异的问题，最终证明下来都是低级错误引起；越诡异，越低级。



  [1]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/chrome_network_canceled/chrome_canceled.png
  [2]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/chrome_network_canceled/chrome_canceled_detail.png
  [3]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/chrome_network_canceled/%E6%8A%93%E5%8C%85.png
