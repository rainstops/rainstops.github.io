---
layout: post
title: Ubuntu安装Jekyll异常
category: "Tool"
---
Ubuntu 14-04 安装 Jekyll，按照GitHub官网给出的安装步骤，使用Bundler安装时出现异常：Failed to build gem native extension.

安装ruby-dev后仍然解决不了问题。

可能是ruby版本的问题，虽然使用ruby -v指令查看ruby版本是1.9.3。但是安装Jekyll是显示的ruby版本为1.9.1。

这种情况下可以使用rvm安装最新Ruby版本.

### 1.安装RVM

安装RVM（参考：[http://rvm.io/](http://rvm.io)）：
{% highlight bash %}
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ curl -sSL https://get.rvm.io | bash -s stable
{% endhighlight %}
载入 RVM 环境（新开 Termal 就不用这么做了，会自动重新载入的）：
{% highlight bash %}
$ source ~/.rvm/scripts/rvm
{% endhighlight %}
检查一下是否安装正确：
{% highlight bash %}
$ rvm -v
{% endhighlight %}

### 2.用RVM安装Ruby环境：
{% highlight bash %}
$ rvm install 2.2.0
{% endhighlight %}
RVM 装好以后，需要执行下面的命令将指定版本的 Ruby 设置为系统默认版本:
{% highlight bash %}
$ rvm 2.2.0 --default
{% endhighlight %}
这个时候你可以测试是否成功：
{% highlight bash %}
$ ruby -v
{% endhighlight %}

### 3.重新安装Jekyll
如果此时正确的显示了Ruby版本,重新安装Bundler和Jekyll。
