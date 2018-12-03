个人博客原文：[Mac开发系列之python多版本和环境管理（pyenv和virtualenv安装配置使用）](http://blog.taohuawu.club/article/47)

系统版本：Mac OS X El Capitan(10.11)
预先安装：homebrew 安装方法：运行ruby脚本：

```shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
pyenv依赖：python 2.5+ , git  

# pyenv安装

推荐使用pyenv-installer这个插件安装pyenv，这种方式安装会多安装几个是实用的插件，比如：
```
pyenv-virtualenv 用于整合virtualenv

pyenv-pip-rehash 用于使用pip安装包之后自动执行rehash

pyenv-update 用于升级pyenv

使用python-installer方式安装：确保你的电脑可以访问Github，然后在终端运行：
```

```shell
curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
```
即可自动安装pyenv，安装完成后，pyenv默认安装地址是：~/.pyenv，但是如果你的系统的环境变量中存在PYENV\_ROOT则会被安装到PYENV\_ROOT目录下(当使用homebrew等其他工具安装后)信息，在你电脑使用的对应的shell的配置文件中添加环境变量（如使用的shell是bash则在~/.bashrc中添加，若是zsh则在~/.zshrc中添加，etc）：
```shell
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```
至此，完成pyenv的安装和基本配置。

**ps:pyenv默认安装地址是~/.pyenv，但是如果你的系统的环境变量中存在PYENV\_ROOT则会被安装到PYENV\_ROOT目录下(当使用homebrew等其他工具安装后)**

# pyenv工作原理：

系统运行python的时候会检测环境变量PATH, 如在以下路径查找：
```shell
/usr/local/bin:/usr/bin:/bin
```
系统会检测/usr/local/bin是否存在python，如果不存在则继续搜索/usr/bin，以及类推 pyenv的工作原理是在PATH中添加shims目录：
```shell
~/.pyenv/shims:/usr/local/bin:/usr/bin:/bin  
```

# 卸载pyenv
pyenv的卸载比较简单，只需要删除对应的目录即可，默认目录是：~/.pyenv。

# pyenv使用

## 查看可安装的包：
```shell
$ pyenv install -l
```

## 查看已安装的python：

**versions和version命令**

versions命令列出你已经安装的Python版本以及当前使用的版本
```shell
pyenv versions
```
如下图：
![](http://static.oschina.net/uploads/space/2015/1216/201418_g23s_2282865.png)
version命令打印你当前使用版本。version命令的输出类似versions命令，但是它只包含了当前版本那一行，并且没有前导的。

版本名称为system代表的是系统预装的Python版本。

安装2.7.11：  
```shell
$ pyenv install 2.7.11
$ pyenv rehash （这一步是更新pyenv中的python版本数据，每次安装新的python版本后需执行此命令。）
```
安装指定版本的python只需使用pyenv install 版本号即可。

卸载python版本：

使用pyenv uninstall {版本号}
```shell
pyenv rehash
```
## 切换当前目录的python版本：

切换和使用指定的版本Python版本有3种方法：
![](http://static.oschina.net/uploads/space/2015/1216/201931_z6ey_2282865.png)

### global和local命令：

global命令和local命令都是用来切换当前Python版本的命令。不同之处在于，global的切换是全局的，而local的切换是局部的。
```shell
pyenv local 2.7.11
```
以上命令：会在当前目录下创建一个.pyenv-version文件，文件内容为2.7.11，pyenv通过这种形式，标记当前目录使用Python-2.7.11。如果其子目录下面没有.pyenv-version文件，那么此版本讲继承到子目录。
```shell
pyenv global 2.7.11
```
以上命令：会修改$PYENV_HOME/version文件的内容，标记全局Python版本，如何理解全局Python版本，可以认为全局版本是一个默认值，当一个目录及其父目录下面都没有.python-version文件的时候，会使用全局版本。

一般的，我们不修改全局版本，而使用期默认值system，因为在unix系统上，很多系统工具依赖于Python，如果我们修改了Python的版本，会造成绝大多数的依赖Python的系统工具无法使用，如果你不小心修改了，也不要紧张，使用global命令修改回来就可以了，有时候，你发现部分系统工具无法使用，你也可以看看你当前的Python版本。

系统全局用系统默认的Python比较好，不建议直接对其操作  
```shell
pyenv global system
```
用local进行指定版本切换，一般开发环境使用。
```shell
pyenv local 2.7.11
```
对当前用户的临时设定Python版本，退出后失效
```shell
pyenv shell 3.5.1
```
取消某版本切换
```shell
pyenv local 3.5.1 --unset
```
优先级关系：shell —> local —> global

  

# 利用virtualenv和Virtaulenvwrapper管理虚拟Python环境

virtualenv用于创建独立的Python环境，多个Python相互独立，互不影响，它能够：

- 在没有权限的情况下安装新套件
- 不同应用可以使用不同的套件版本
- 套件升级不影响其他应用

Virtaulenvwrapper是virtualenv的扩展包，用于更方便管理虚拟环境，它可以做：

- 将所有虚拟环境整合在一个目录下
- 管理（新增，删除，复制）虚拟环境
- 切换虚拟环境


## 1、安装和使用方法

### 安装
```shell
pip install virtualenv
pip install virtualenvwrapper
```
### 创建虚拟环境
```shell
mkvirtualenv \[虚拟环境名称\]
```
此时还不能使用virtualenvwrapper，默认virtualenvwrapper安装在/usr/local/bin下面，实际上你需要运行virtualenvwrapper.sh文件才行，先别急，打开这个文件看看,里面有安装步骤，我们照着操作把环境设置好。

如果你使用的bash或者zsh

### 创建目录用来存放虚拟环境
```shell
mkdir $HOME/.virtualenvs
```
**在. bash_profile 或者 .zshrc 追加两行：**
```shell
export WORKON_HOME=$HOME/.virtualenvs
source /usr/local/bin/virtualenvwrapper.sh
```
运行： 
```shell
source ~/.bashrc
source ~/.zshrc
```
此时virtualenvwrapper就可以使用了。

## 2、创建虚拟环境例如：

### 在当前的环境的Python版本下创建名称为py3dev的虚拟环境。
```shell
mkvirtualenv py3dev
```
默认情况下，虚拟环境会依赖系统环境中的site packages，就是说系统中已经安装好的第三方package也会安装在虚拟环境中，如果不想依赖这些package，那么可以加上参数 –no-site-packages建立虚拟环境

例如：
```shell
mkvirtualenv --no-site-packages \[虚拟环境名称\]
mkvirtualenv --no-site-packages py3dev
```

## 3、查看创建的虚拟环境:
```shell
λ ~/ lsvirtualenv

py2dev

======

py3dev

======

λ ~/ workon

py2dev

py3dev
```
## 4、启动某虚拟环境：
```shell
works \[虚拟环境名称\]
workon py3dev
```

## 5、删除某个虚拟环境：
```shell
rmvirtualenv \[虚拟环境名称\]
rmvirtualenv py3dev
```

## 6、删除某个虚拟环境需要先退出这个环境：
```shell
deactivate  
```

# 融合 pyenv、virtualenv、Virtaulenvwrapper 管理Python版本和虚拟环境

如何对Python2.7.11和Python3.5.1版本分别创建虚拟环境？

有这三个工具其实非常简单，主要是确保环境切换成功后在创建虚拟环境。

**为确保切换成功，我建议 source .zshrc 一下在切换（如果是使用pyenv-installer安装的，则不需要，若是直接安装的pyenv则需要，具体原因仍不清楚。）**

```shell
#安装全新的Python2.7.11版本
pyenv install 2.7.11
pyenv rehash

#切换到刚安装的这个版本
pyenv local 2.7.11
#确保切换成功
source .zshrc
#验证一下版本,pip发现里面包很少
pip list
#验证版本
python -V
#务必在这个新的2.7.11中安装
pip install virtualenv
pip install virtualenvwrapper
#务必
source .zshrc
#创建2.7.11的开发环境
mkvirtualenv py2dev
#创建完某版本的开发环境后务必退出，当前虚拟环境，不然就是虚拟环境中在创建了。
deactivate
#退出2.7.11环境
pyenv local --unset 2.7.11
source .zshrc
```

3.5.1的虚拟环境创建也是一样，因此验证2.7.11和3.5.1的虚拟环境

```shell
(py3dev) ~  deactivate
~  workon py2dev
(py2dev)~  python -V
Python 2.7.11
(py2dev)~  deactivate
~  workon py3dev
(py3dev)~  python -V
Python 3.5.1
(py3dev)~  deactivate
~
```

#愉快无痛升级，一键升级所有PIP包  
```shell
pip list --outdated | grep '^\[a-z\]* (' | cut -d " " -f 1 | xargs pip install -U
```

# pyenv与virtualenv

pyenv通过插件，可以很好的和virtualenv一起工作，通过整合virtualenv，pyenv实现了真正意义上的环境隔离，每个项目都相当于使用一个单独的解释器。

通过pyenv-installer安装的pyenv，已经安装好virtualenv插件了，如果不是通过pyenv-installer安装的pyenv，你可能需要自己安装virtualenv插件，安装方法也很简单：
```shell
cd $PYENV_ROOT/plugins
git clone https://github.com/yyuu/pyenv-virtualenv.git
```
直接把插件clone下来就安装完成了。
安装完成之后，我们可以通过virtualenv命令即可创建虚拟环境，virtualenv的一般用法如下：
```shell
pyenv virtualenv \[-f|--force\] \[-u|--upgrade\] \[VIRTUALENV_OPTIONS\] <version> <virtualenv-name>
```
选项-f表示强制的，也就是如果已经存在这个虚拟环境，那么将会覆盖这个虚拟环境 选项-u表示upgrade，用于修改已经存在的虚拟环境的Python版本 VIRTUALENV_OPTIONS 是传递给virtualenv的选项，可以通过virtualenv的帮助获取选项的含义 version 表示Python版本 virtualenv-name 是我们给虚拟环境指定的名字

例如:
```shell
pyenv virtualenv 2.7.11 my_project
```
以上命令就创建了一个基于Python-2.7.11,名为my_project的虚拟环境。创建好的虚拟环境犹如一个单独Python版本一样，我们可以通过local或者global命令切换过去。

由于每个解释器间是完全隔离的，所以强烈建议你的每个项目，都放置在单独的虚拟环境中。

virtualenv插件还提供了virtualenvs命令，用于列出所有已经创建的虚拟环境，
```shell
pyenv virtualenvs
```
以上命令列出我们所有已经创建的虚拟环境，已经虚拟环境基于那个Python版本。
当我们的一个项目生命周期结束的时候，我们或许会想要删除虚拟环境以释放我们的硬盘空间，删除虚拟环境非常简单，直接用uninstall命令像删除正常的Python版本一样就可以了。

事实上，虚拟环境一旦创建，你就可以把他当成一个独立的版本来使用和维护了。

删除、设置当前目录python以及取消设置版本使用下面的命令
```shell
# 创建一个基于`python 2.7.10`版本的名字叫做`env27`的虚拟环境, 使用`--no-site-packages`参数来隔离系统的site-packages
$ pyenv virtualenv 2.7.10 env27 --no-site-packages
# 指定目录使用evn27的虚拟环境
$ cd code
$ pyenv local env27
# 取消设定
$ pyenv local env27 --unset
# 删除
$ pyenv uninstall env27
```