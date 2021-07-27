---
layout: article
title: Configure TeXt Jekyll theme
author: Ciao
description: TeXt Jekyll 主题配置备忘录
categories:
 - tech
tags:
 - tech
 - Blog
 - github
 - theme
key: page-article
---
> TeXt 主题设置排坑备忘录

等我想起来这个博客的时候，当时的NEXT主题已经不太兼容我的本地环境了(MacOS M1)，所以在更新了`ruby3`版本并比较了目前流行的各种主题后，发现这个[TeXt Theme](https://jamstackthemes.dev/demo/theme/jekyll-text-theme/)还不错。但对于我这种远离网页IT前后端的人来说，安装和配置的过程依然十分痛苦，需要仔细记录一下以便日后查询。

# S1.安装主题
>参考[TeXt Theme Docs](https://jamstackthemes.dev/demo/theme/jekyll-text-theme)

安装过程比较顺利。由于我拢共也没更新多少帖子，直接克隆覆盖我的旧仓库就行了
```sh
git clone https://github.com/kitian616/jekyll-TeXt-theme.git scelta.github.io/
cd scelta.github.io/
git remote set-url origin git@github.com:Scelta/scelta.github.io.git
```

打开`_config.yaml`，修改一些个性化设置。

# S2.更新jekyll配置

### 更新ruby
> 参考[asdf Getting Started](https://asdf-vm.com/guide/getting-started.html#_1-install-dependencies),进行升级。

虽然MacOS自带ruby，但首先它的版本比较低，同时每次升级安装又需要root权限，并不方便。我找了[asdf](https://asdf-vm.com)进行ruby版本管理并安装到我个人用户目录下。


### 安装jekyll
> 参考[jekyll docs](https://jekyllrb.com/docs/), 安装jekyll

在完成初步配置，进行本地预览：
```shell
bundle exec jekyll serve
```
此时可能会出现gem plugin版本报错。可能是这个主题最近没有进行相应更新，根据提示进行了版本强制升级。

在`jekyll-text-theme.gemspec`中添加：
```
spec.add_dependency "rake", ">= 12.3.3"
```
在`Gemfile.lock`中找到`rake`并修改它的版本为：
```
rake (>= 12.3.3)
```
在`Gemfile`中添加：
```
gem "rake", ">=12.3.3"
```

然后根据该主题的语法修改了我的几篇文章的yaml header，终于成功渲染了本地网页预览。确认ok后push到了github上部署了本博客。

# S3.Troubleshooting

在配置留言模块的时候，我选择了该主题提供的`gitalk`方法。因为我十分不了解网页代码调用的方式，这部分折腾了好久。
我查阅了gitalke的页面，很多地方并不明白是什么意思，但现在我大致悟了。需要做的事情如下：

### 网页端模块配置
参考[gitalk仓库说明文档](https://github.com/gitalk/gitalk#install)。由于`TeXt`主题已经内置了这部分，所以可以直接跳过这一步。

### 授权应用创建
应该是需要登录github获取账号进行留言，这个步骤是需要认证的，所以需要在github上[Register a new OAuth application](https://github.com/settings/applications/new)。
此处需要填写的内容真的让我绞尽脑汁，无论是gitalk的文档还是[github的文档](https://docs.github.com/v3/oauth/)都没能让我明白是啥意思。关键是每次尝试都需要push一次才能看到效果，十分麻烦。
最终，经过一些搜索和依葫芦画瓢，我发现两个url可以填写相同，并指向我现在博客的域名上。如下：
```
Application name: gitalk
Homepage URL: http://biochao.cc
Application description: gitalk for my blog
Authorization callback URL: http://biochao.cc
```
确认并跳转后，从页面上生成secret ID。将这个ID和 clientID记录到`_config.yaml`相应的部分。

### 解决403问题
在我部署好网页，从评论模块尝试登录时，发现返回403错误。在相应帖子中（如https://github.com/gitalk/gitalk/issues/433）发现是中间用到的一个反向代理居然是人家的一个demo，在被滥用后限制了转发数，所以现在基本处于一个半瘫痪状态。现在需要利用gitalk的proxy参数制定一个新的反向代理才行。这我哪懂啊……
此时我参考了[一篇类似的博客](https://zhuanlan.zhihu.com/p/350735142)也在解决这个问题。但对方用的是Hexo模版，调用gitalk的方式跟我这个主题不太相同，TeXt也没有提供proxy的接口。但这给了我去折腾的勇气。
### 自建反向代理
首先，根据[stackoverflow.com](https://stackoverflow.com/questions/47076743/cors-anywhere-herokuapp-com-not-working-503-what-else-can-i-try)上的答案，自己搭建一个反向代理。照着几分钟就能完成。
1. 登录[https://dashboard.heroku.com](https://dashboard.heroku.com)注册账号
2. 在网页端创建一个app，如命名为`biochao-gitalk`。
3. 根据提示安装Heroku CLI，一个命令行工具。
4. 打开终端
```shell
git clone https://github.com/Rob--W/cors-anywhere.git biochao-gitalk
cd biochao-gitalk
heroku login -i #登录。如遇到报错可以先把代理关掉,unset http_proxy等之后再尝试
heroku git:remote -a biochao-gitalk
npm install #若没有npm就通过brew install npm进行安装
heroku create
git commit -am "make gitalk"
git push heroku master
```
这个时候终端会返回给你一个网址，告诉你部署成功：
```
remote:        https://biochao-gitalk.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
To https://git.heroku.com/biochao-gitalk.git
 * [new branch]      master -> master
```
记住这个remote地址。如果你看到了我这篇文章，你也可以直接用这个地址，我觉得应该是通用的。但其实自己部署一个也很方便，免费快捷。

### `TeXt`实现`gitalk`的`proxy`自定义接口
经过一系列门外汉尝试，我发现可以通过以下修改来实现：
1. 在`_config.yaml`中找到`gitalk:`行，添加一个子类(见下面最后一行)
```yaml
owner       : Scelta # GitHub repo owner
admin: Scelta # GitHub repo owner and collaborators, only these guys can initialize GitHub issues, IT IS A LIST.
  # - your GitHub Id
proxy: https://biochao-gitalk.herokuapp.com/https://github.com/login/oauth/access_token #添加这行
```
对于直接用我这个地址的同学可以将后半段https的地址改为你的域名进行测试。

2. 在`_includes/comments-providers/gitalk.html`中如下位置添加一行，但其实我并不懂这些语法，只是改了变量名而已。

{% raw %}
```
  {% assign _admin = _admin | slice: 2, _last %}
  {% assign _proxy = _proxy | slice: 2, _last %}  #紧跟着增加这行
```
{% endraw %}
> If this chunk somehow disappeared, ref: [http://shopify.github.io/liquid/tags/template/](http://shopify.github.io/liquid/tags/template/)

在相同文件的后续段落中继续添加一行：
```
				admin: [{{ _admin }}],
				id: '{{ page.key }}',
				proxy: '{{ site.comments.gitalk.proxy }}' #紧跟着增加这行
```
3. 保险起见，我在`_data/variables.yml`中找到`gitalk`段落，更新了其中的`js`和`ccs`版本：
```
gitalk:
  js: 'https://unpkg.com/gitalk@latest/dist/gitalk.min.js'
  css: 'https://unpkg.com/gitalk@latest/dist/gitalk.css'
```

至此，经过测试，gitalk终于活了。

### 为什么我的博文末尾没有评论模块？
在折腾的过程中，我在`_includes/comments-providers/gitalk.html`的开头看到这样一段代码：

```html
{\%- if page.key and
  site.comments.gitalk.clientID and
	site.comments.gitalk.clientSecret and
	site.comments.gitalk.repository and
	site.comments.gitalk.owner and
	site.comments.gitalk.admin -\%}
```
反复研究后我发现，`page.key`应该是文档yaml header定义的`key`变量,`site.comments.gitalk.*`则是`_config.yaml`中`gitalk`的参数。也就是只有当这些变量和参数都存在时，这个模块才会生效。而该主题模版提供的文章模版中，并不存在`key`,所以评论模块无法生效。对比了`about.html`文件后我发现，只需要在yaml header中添加`key:`即可：
```
key: page-article  #只需要出现key，值可以自定义
```
顺便提交了一个[issue](https://github.com/kitian616/jekyll-TeXt-theme/issues/348)给作者，希望能尽快出官方接口。


至此，该模版的问题基本都解决了。后续若出现新问题我再继续更新。
