---
layout: post
title: centos 搭建jekyll静态博客
category: [Linux]
tags: [Linux]
---

#####话说为什么要使用静态博客,wordpress太过臃肿,而且越来越多相对无用的功能,主要想搭建一个专心写博客,不用折腾的环境,之前我得博客一直是放在github上得..github有一个git-page用来给开源项目构建主页的.主要是使用了jekyll server,永久免费,而且还能抵挡大流量的攻击(前提是你有那么高流量).所以一直用了下来.但是github前段时间一直不稳定.因此迁移回了国内.

这次记录下jekyll server的部署的一个过程.

#### 其他静态博客框架可以查看这个地址<https://www.staticgen.com/>

####环境:
#####aliyun 服务器, centos 7.0 64位

* 1.安装ruby, 直接使用rvm安装ruby,ruby版本很混乱.所以不要轻易升级..但是一定要安装2.0之后的版本.2.0ruby优化了很多GC方面得东西.提升了不少性能.另外2.0之后会默认安装gem包管理.不用单独再去安装.可以节省很多功夫.

--- rvm 链接: <https://www.rvm.io/>
安装ruby 可以使用 `curl -sSL https://get.rvm.io | bash -s stable`

---如果不能下载,请试试这边 <https://www.rvm.io/rvm/install> 应该比较简单易懂了.

---rvm 安装完毕之后.使用`rvm -v `查看是否安装成功

#####题外话: 如果你使用centos 6.x的系统,那么使用`yum install -y ruby `最多只能更新到ruby 1.8.x

* 2.rvm安装完成之后开始安装ruby默认可以使用版本号来安装.你想要的ruby版本.例如 `rvm install 2.1.1 `如果使用`rvm install ruby` 默认安装最新版

---你可以使用`rvm list known`来查看所有得资源以及ruby的版本

---安装完成之后使用`rvm list `查看已经安装得包.如果你想安装多个版本的ruby,那么可以直接安装,多版本之间进行切换的话,使用

---`rvm 2.1.1 --default` 来切换.

---ruby 安装完成之后,使用`ruby -v `来验证是否安装成功

---如果你安装得是2.0.0 或者之后的版本,那么可以直接使用gem包管理.验证gem是否可用.可以使用gem -v 来查看.

####如果你想要更快速的安装.那么要切换下gem 的源.

1.使用gem source list 查看已经拥有的源.你会发现是一个rubygems.org的源.国内安装会很慢.

2.使用gem source -r https://rubygems.org/  移除这个源

3.然后使用gem sources -a https://ruby.taobao.org/ 添加淘宝的源

4.最后使用gem sources -l 查看已经加入的源.

5.开始安装 gem没有问题.开始安装bundle. 命令: `gem install bundle`

一般安装这个命令的时候中途可能出现依赖缺失.你只需要例如这样得:

```c

Gem::Ext::BuildError: ERROR: Failed to build gem native extension.
    /usr/local/rvm/rubies/ruby-2.0.0-p643/bin/ruby -r ./siteconf20150422-19711-1x7hpk.rb extconf.rb
Cannot allocate memory - /usr/local/rvm/rubies/ruby-2.0.0-p643/bin/ruby -r ./siteconf20150422-19711-1x7hpk.rb extconf.rb 2>&1

Gem files will remain installed in /usr/local/rvm/gems/ruby-2.0.0-p643/gems/nokogiri-1.6.6.2 for inspection.
Results logged to /usr/local/rvm/gems/ruby-2.0.0-p643/extensions/x86_64-linux/2.0.0/nokogiri-1.6.6.2/gem_make.out
An error occurred while installing nokogiri (1.6.6.2), and Bundler cannot
continue.
Make sure that `gem install nokogiri -v '1.6.6.2'` succeeds before bundling.

```
#####那么你只需要依次去安装就好了.执行 `gem install nokogiri -v '1.6.6.2' `命令.执行完成之后在继续执行 `gem install bundle`直到解决bundle的安装.

* 3.安装结束之后.安装 jekyll, `gem install jekyll `

这个过程一般不会遇到什么问题.最后使用`jekyll -v `来验证安装

####进入目录文件夹,创建Gemfile 文件,然后输入

```c
source 'http://ruby.taobao.org/'
gem 'execjs'
gem 'therubyracer'
gem 'github-pages'

```

####bundle 会根据Gemfile配置文件自动加载安装相关配置,然后执行`bundle install`

####最后环境安装结束.创建文件夹.然后进入文件夹,将某个静态项目clone下来.例如本博客的项目

####git clone https://github.com/woooowen/woooowen.github.io.git

#####clone结束之后.执行命令来开启server服务 `bundle exec jekyll serve`

#####运行成功.访问他给你的提示地址来登录访问.

that's all .我只记录个大概过程.详细的经过请自己去折腾吧...运维真辛苦..
