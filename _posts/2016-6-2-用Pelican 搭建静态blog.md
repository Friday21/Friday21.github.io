---
layout:     post
title:      "用Pelican 搭建静态blog"
date:       2016-06-01
author:     "FridayLi"
catalog: true
tags:
  - Python
---

# 用Pelican 搭建静态blog
之所以选用Pelican而不是WordPress, 是因为前者是用Python写的，而后者则是世界上最好的语言， 目前我对最好的语言还不太感兴趣，所以根据网上教程用Pelican搭建了我的第一个博客。 不过现在本博客是用Flask + Bootstrap + Mysql搭建的， 给了我更多的灵活性， 也更加美观一点

## 搭建环境
Ubuntu 14.04 LTS
Pelican 3.3.0
Apache 2.4.0
VPS：  digital ocean SFO（非必须，可已选择github page）
域名解析：Godaddy

## 环境配置
安装pelican
`pip install pelican`  

创建blog目录  
`cd /var/www`  
`mkdir blog`  
`pelican-quickstart`
之后终端会问几个问题，一路默认回车就行（有一个要填一下，不过这些都可以在后来生成的pelicanconf.py文件中修改的）  
生成的目录结构如下：
```python  
blog/
├── content                # 存放输入的markdown或RST源文件
│   └── (pages)            # 存放手工创建的静态页面，可选
│   └── (posts)            # 存放手工创建的文章，可选
├── output                 # 存放最终生成的静态博客
├── develop_server.sh      # 测试服务器
├── Makefile               # 管理博客的Makefile
├── pelicanconf.py         # 配置文件
└── publishconf.py         # 发布文件，可删除
```
## 主题和插件
* 克隆主题到本地
`git clone https://github.com/getpelican/pelican-themes.git`
* 安装主题——bootstrap3
`cd pelican-themes`
`pelican-themes -i pelican-bootstrap3`
这一步是将主题`pelican-bootstrap3`安装到Python库里，Ubuntu下路径为`/usr/local/lib/python2.7/dist-packages/pelican/themes/`,所以后面修改字体和banner时需要在这个文件夹下的`pelican-bootstrap3/templates`文件修改
* 克隆插件到本地
`cd /var/www/blog`
`git clone git://github.com/getpelican/pelican-plugins.git`
此处将pelican的插件放在`/var/www/blog`的文件夹下
## 配置pelicanconf.py
博客的很多内容都是在`/var/www/blog/pelicancof.py`文件中配置的
``` python
AUTHOR = u'Friday'
SITENAME = u'\u6211\u7684\u7cbe\u795e\u5bb6\u56ed' #博客名字
SITEURL = 'http://localhost:80' 
TIMEZONE = 'Asia/Shanghai' #时区改成上海
THEME = 'pelican-bootstrap3' #主题设置为pelican-bootstrap3

#添加社交账号
SOCIAL = (('facebook', 'https://www.facebook.com/li.dongyong?ref=bookmarks'),
          ('twitter','https://twitter.com/dongyongli'),
          ('github','https://github.com/Friday21'),)

DEFAULT_PAGINATION = 5 #每页显示5篇文章

PLUGIN_PATHS = [u'pelican-plugins',] #插件地址，因为插件放在了和blog同级目录，所以可以       直接用pelican-plugins, #若放在其他地方改成相应路径即可
PLUGINS = ['sitemap', 'related_posts', 'random_article',
           'liquid_tags.img', 'liquid_tags.video',
           'liquid_tags.youtube', 'liquid_tags.vimeo',
           'liquid_tags.include_code','tag_cloud','tipue_search']#使用到的插件
RANDOM = 'random.html'
RELATED_POSTS_MAX = 10
DIRECT_TEMPLATES = ('index', 'categories', 'authors', 'archives', 'tags','search')
ARTICLE_URL = 'blog/{slug}.html'
ARTICLE_SAVE_AS = 'blog/{slug}.html'
PAGE_URL = '{slug}.html'
PAGE_SAVE_AS = '{slug}.html'
TAG_URL = 'tags/{slug}.html'
TAG_SAVE_AS = 'tags/{slug}.html'
TAGS_URL = 'tags.html'
DISPLAY_CATEGORIES_ON_MENU = False
DISPLAY_CATEGORIES_ON_SIDEBAR = False #不显示categories（感觉有tags就够了）
DISPLAY_TAGS_ON_SIDEBAR = True #在边上显示标签栏
```
## 发布第一篇博客
用markdown写下第一篇文章，开头要包含以下内容：
```python
Title: My super title 必须
Date: 2010-12-03 10:20 必须
Modified: 2010-12-05 19:30
Category: Python
Tags: pelican, publishing
Slug: my-super-post 必须
Authors: Alexis Metaireau, Conan Doyle
Summary: Short version for index and feeds

```
写好后把md文件放在blog下的content目录中，发布博客
`cd /var/www/blog`
`make publish`
如果成功的话可以在blog目录下看到output文件夹，里面就是网站的所有内容，将其部署到github pages上即可以访问，在github创建username.github.io repo，其中username为你github的用户名，然后在Ubuntu上把博客内容推送到github pages上
```
cd /var/www/blog/output
git init
git add .
git commit -m'first commit'
git remote add origin git@github.com:username/username.github.io.git
git push -u origin master
```
成功后访问username.github.io即能成功看到你的blog

##Apache 配置
如果不想用github page当然也可以用自己的VPS，让blog运行在Apache上
* 安装Apache
`sudo apt-get install apache2`
* 配置虚拟主机
`cd /etc/apache2/sites-available/`
新建blog的配置文件
`vim blog.conf` 
添加如下内容
```python
<Directory /var/www/blog/output> #博客目录
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
<VirtualHost *:80>
    # Admin email, Server Name (domain name) and any aliases
    ServerAdmin lidongyong22@gmail.com 
    ServerName  fridayhaohao.com #服务器名字
    ServerAlias www.fridayhaohao.com #域名

    # Index file and Document Root (where the public files are located)
    DirectoryIndex index.php index.html
    DocumentRoot /var/www/blog/output #文件目录
</VirtualHost>

```
使文件生效：
```
$ sudo a2ensite duncanlock.test
$ sudo service apache2 restart
```
成功后需要在godaday上将你的域名连接到服务器上的ip， 在DNS ZONE FILE下添加一天A记录，HOST：@，POINTS TO: 你的服务器IP，等待一段时间后就可以通过你的域名来访问你的blog啦！

## 进一步完善博客
### bootstrap主题
bootstrap下还细分有很多主题，免费的可以在[这里](http://bootswatch.com/)看到，我选的是Cerulean, 舒服的蓝色主题，只需要在pelicanconf.py中添加`BOOTSTRAP_THEME = 'cerulean'`, 然后在blog目录下make publish 就会生效了  

### banner
根据DandyDev的pelican-bootstrap3的[文档](https://github.com/DandyDev/pelican-bootstrap3)的说明：
>A banner image can be added to the theme, displayed with the SITENAME and an optional subtitle. Config options are as follows:
Set the banner image with BANNER = '/path/to/banner.png'
Set the subtitle text with BANNER_SUBTITLE = 'This is my subtitle'
By default, the banner is only shown on the index page. To display the banner on all pages, set BANNER_ALL_PAGES = True  


在content目录下新建文件夹`images`, 把要设置的首页头部图片放在此文件夹下，并在pelican.conf中添加 `BANNER = '/images/banner.png'`, banner.png 为图片的文件名，建议选好想要的图片后调整到适合的长宽比，用截图工具就行。我选了一张大海的图片，与上一步选的主题配合的很好。效果如下图：  
![描述](/img/old-post/b21d0e37914cf0ce1d75e3da174064cd.PNG)   


### 字体
pelican默认英文字体，所以中文显示很难看！按照习惯还是调成微软雅黑舒服一点，办法如下：
尽如Python的pelican库文件夹，编辑style.css文件我的目录是`/usr/local/lib/python2.7/dist-packages/pelican/themes/pelican-bootstrap3/static/css/style.css`
在头部添加以下两段：
```css
h1, h2, h3, h4, h5, h6, .h1, .h2, .h3, .h4, .h5, .h6 {
    font-family: "Microsoft YaHei UI", "Microsoft YaHei", "Arial", "Verdana", "Tahoma";
    font-weight: 500;
}

body {
    font-family: "Microsoft YaHei", "Arial", "Verdana", "Tahoma";
    color: #424242;
}
```
make publish 后就能看到中文显示舒服多了。
其实在这个文件中还可以进一步美化banner，比如把我的精神家园后面的黑框改为透明的，只需要把backgroud中的0.7改为0就行啦
```css
#banner .copy {
    background: none repeat scroll 0 0 rgba(0, 0, 0, 0.7);
    display: inline;
    float: left;
    max-width: 600px;
    padding: 20px;
    position: relative;
    z-index: 1;
}
```
改完后make publish就能看到效果了，体会下两张图的不同：

![描述](/img/old-post/728b364a7be37cdee71c2c254dde7141.PNG)   



下图是设置为0.7的效果，上图是设置成0的透明效果。哈哈虽然前端基础为0，但这个过程中对前端增加了些了解。

### 添加多说评论
作为blog怎么能少的了评论呢，pelican自带的评论插件是Disqus comments， 但那是针对国外用户的，咱在局域网中上不了Facebook和Twitter，所以还是选择国内有名的评论插件——多说评论。  
首先需要在[多说](http://duoshuo.com/)的网站上注册，绑定网站信息，获取js代码，并根据提示修改成如下代码：
```html
<!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="{{ article.slug }}" data-title="{{ article.title }}" data-url="{{ SITEURL }}/{{ article.url }}"></div>
<!-- 多说评论框 end -->
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
var duoshuoQuery = {short_name:"lyon0804"};
    (function() {
        var ds = document.createElement('script');
        ds.type = 'text/javascript';ds.async = true;
        ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
        ds.charset = 'UTF-8';
        (document.getElementsByTagName('head')[0]
         || document.getElementsByTagName('body')[0]).appendChild(ds);
    })();
    </script>
<!-- 多说公共JS代码 end -->
```
尽如pelican的Python库，编辑评论模板，  
`vim /usr/local/lib/python2.7/dist-packages/pelican/themes/pelican-bootstrap3/templates/includes/comments.html`
对照着disqus评论添加如下内容：
```html
{% if DUOSHUO_SITENAME %}
    <hr/>
    <section class="comments" id="comments">
        <h2>Comments</h2>
        <div class="ds-thread" data-thread-key="{{ article.slug }}" data-title="{{ article.title }}" data-url="{{ SITEURL }}/{{ article.url }}"></div>
        <script type="text/javascript">
            var duoshuoQuery = {short_name:"{{ DUOSHUO_SITENAME }}"};
            (function() {
                var ds = document.createElement('script');
                ds.type = 'text/javascript';ds.async = true;
                ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
                ds.charset = 'UTF-8';
                (document.getElementsByTagName('head')[0]
                 || document.getElementsByTagName('body')[0]).appendChild(ds);
            })();
        </script>
        <noscript>Please enable JavaScript to view the <a href="http://duoshuo.com/">comments powered by
        Duoshuo.</a></noscript>
        <a href="http://duoshuo.com" class="dsq-brlink">comments powered by <span>Duoshuo</span></a>

    </section>
{% endif %}
```
然后在pelicanconf.py中添加DUOSHUO_SITENAME字段，make publish 即可生效，我在多说网站上填的是fridayhaohao，所以我添加了`DUOSHUO_SITENAME = 'fridayhaohao'`


###Google站内搜索
pelican自带的tipue_search用起来bug多多，果断换成高大上的Google站内搜索 ，根据网上的指导自己怎么做都不成功，最终折腾的结果是在pelicanconf中启用tipue_search（因为我懒得去调搜索框的大小），但是把search.html内容换成Google站内搜索的代码，搞定！(pelican-bootstrap3似乎没有bootstrap2的Google搜索内置代码，坑！)
* 启用tipue_search
在PLUGINS中添加tipue_search
在DIRECT_TEMPLATES中添加search
```python
PLUGINS = ['sitemap', 'related_posts', 'random_article',
           'liquid_tags.img', 'liquid_tags.video',
           'liquid_tags.youtube', 'liquid_tags.vimeo',
           'liquid_tags.include_code','tag_cloud','tipue_search']
DIRECT_TEMPLATES = ('index', 'categories', 'authors', 'archives', 'tags','search')
```
* 换成Google站内搜索
首先在[Google站内搜索](https://cse.google.com/cse/all)申请，得到自己的ID，然后把search.html替换为如下内容：
```html
<!DOCTYPE html>
<html lang="zh_CN">
<head>
<meta charset="utf-8">
<title>站内搜索</title>
</head>
  <body>
<style>
#search-box {
    position: relative;
    width: 50%;
    margin: 0;
    padding: 1em;
}

#search-form {
    height: 30px;
    border: 1px solid #999;
    -webkit-border-radius: 5px;
    -moz-border-radius: 5px;
    border-radius: 5px;
    background-color: #fff;
    overflow: hidden;
}

#search-text {
    font-size: 14px;
    color: #ddd;
    border-width: 0;
    background: transparent;
}

#search-box input[type="text"] {
    width: 90%;
    padding: 4px 0 12px 1em;
    color: #333;
    outline: none;
}
</style>
<div id='search-box'>
  <form action='/search.html' id='search-form' method='get' target='_top'>
    <input id='search-text' name='q' placeholder='Search' type='text'/>
  </form>
</div>
<div id="cse" style="width: 100%;">Loading</div>
<script src="http://www.google.com/jsapi" type="text/javascript"></script>
<script type="text/javascript"> 
  google.load('search', '1', {language : 'zh-CN', style : google.loader.themes.V2_DEFAULT});
  google.setOnLoadCallback(function() {
    var customSearchOptions = {};  var customSearchControl = new google.search.CustomSearchControl(
      '012191777864628038963:**********<!写入你申请的google站内搜索的ID号>）', customSearchOptions);
    customSearchControl.setResultSetSize(google.search.Search.FILTERED_CSE_RESULTSET);
    var options = new google.search.DrawOptions();
    options.enableSearchResultsOnly(); 
    customSearchControl.draw('cse', options);
    function parseParamsFromUrl() {
      var params = {};
      var parts = window.location.search.substr(1).split('\x26');
      for (var i = 0; i < parts.length; i++) {
        var keyValuePair = parts[i].split('=');
        var key = decodeURIComponent(keyValuePair[0]);
        params[key] = keyValuePair[1] ?
            decodeURIComponent(keyValuePair[1].replace(/\+/g, ' ')) :
            keyValuePair[1];
      }
      return params;
    }

    var urlParams = parseParamsFromUrl();
    var queryParamName = "q";
    if (urlParams[queryParamName]) {
      customSearchControl.execute(urlParams[queryParamName]);
    }
  }, true);
</script>
</body>
</html>
```
make publish 生效，但是**局域网**内上不了Google，所以搜索功能只能翻墙使用，但对我来说不算问题，blog搜索功能主要还是自己用

### 参考文章
1. [谷歌站内搜索](http://guozhongxin.com/pages/2014/09/25/build_blog_with_pelican.html)
2. [添加多说评论](http://www.lyon0804.com/zai-pelicanzhong-shi-yong-duo-shuo.html)
3. [字体设置](http://edi.wang/Post/2013/12/9/bootstrap-3-usage-experience)
4. [pelicanconf.py配置](http://blog.atime.me/note/pelican-setup-summary.html)
5. [pelican搭建教程](http://www.lizherui.com/pages/2013/08/17/build_blog.html)
6. [pelican搭建教程2](http://www.jianshu.com/p/d80a5cefc128)
7. [apache2配置](http://duncanlock.net/blog/2013/05/17/how-i-built-this-website-using-pelican-part-1-setup/)




