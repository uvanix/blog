使用rvm管理ruby版本，参考：https://rvm.io/

```
本例使用 rmv 进行 ruby 的安装，可以快捷的切换 ruby 环境。具体可以去 rvm.io 查看。
 
$ gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
$ \curl -sSL https://get.rvm.io | bash -s stable
载入 rvm 环境
 
$ source ~/.rvm/scripts/rvm (root安装：source /usr/local/rvm/scripts/rvm)
安装之后检查一下
 
$ rvm -v
 
list一下可以安装的版本
$ rvm list known
 
再用 rvm 安装 ruby
$ rvm install 2.4.1
 
安装完成之后，设置默认 ruby 的版本
 
$ rvm 2.4.1 --default
 
检查一下 ruby 的版本
$ ruby -v
 
检查一下 gem 的版本
$ gem -v
```


ruby+gem常用命令
```
ruby -v #查看ruby 版本
ruby -e ''require"watir"; puts Watir::IE::VERSION''　#查看watir版本
 
gem -v #gem版本
gem sources -l 列出安装源
gem sources -a XXX 添加安装源
gem sources -r XXX 删除安装源
gem sources -u 更新安装源缓存
gem update #更新所有包
gem update cocoapods #更新某包
gem update --system #更新RubyGems软件自身
gem install rake #安装rake,从本地或远程服务器
gem install rake --remote #安装rake,从远程服务器
gem install watir -v(或者--version) 1.6.2#指定安装版本的
gem uninstall rake #卸载rake包
gem list --local #列出本地包
gem list --remote #列出远程包
gem list --remote #列出可用的gem
gem list d #列出本地以d打头的包
gem rdoc --all #为所有的gems创建RDoc文档
gem fetch mygem #下载一个gem，但不安装
gem search STRING --remote #从可用的gem中搜索
gem query -n ''[0-9]'' --local #查找本地含有数字的包
gem search log --both #从本地和远程服务器上查找含有log字符串的包
gem search log --remoter #只从远程服务器上查找含有log字符串的包
gem search -r log #只从远程服务器上查找含有log字符串的包
gem help #提醒式的帮助
gem help install #列出install命令 帮助
gem help examples #列出gem命令使用一些例子
gem build rake.gemspec #把rake.gemspec编译成rake.gem
gem check -v pkg/rake-0.4.0.gem #检测rake是否有效
gem cleanup #清除所有包旧版本，保留最新版本
gem contents rake #显示rake包中所包含的文件
gem dependency rails -v 0.10.1 #列出与rails相互依赖的包
gem environment #查看gem的环境
```

