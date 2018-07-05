## rails-vagrant

### version info
- Mac OS 10.11.6（El Capitan）
- [Vagrant](https://www.vagrantup.com/intro/index.html) 2.1.1
    - plugins
        - vagrant-vbguest (0.15.2)
- [Virtualbox](https://www.virtualbox.org/wiki/Downloads) 5.2.12
- CentOS 7
  - rbenv 1.1.1-37-g1c772d5
    - ruby 2.5.1p57
  - nginx version: nginx/1.15.0
  - psql (PostgreSQL) 11beta1
- rails 5.2.0

#### other info
- Qiita:[VagrantとVirtualBoxでCentOSにnginxとpumaのRails環境を構築する \- Qiita](https://qiita.com/2no553/items/bc786fd1920312353e7e)
- blog:[VagrantとVirtualBoxでCentOSにnginxとpumaのRails環境を構築する – Ninolog](https://ninolog.com/set-rails-virtualbox-vagrant-with-puma-nginx/)

### setting vagrant

```
$ brew cask install vagrant virtualbox
$ brew cask list
```
```
$ git clone https://github.com/2no553/rails-vagrant.git
$ cd rails-vagrant
$ vagrant up
```

### setting centos

```
$ vagrant ssh
$ sudo yum update -y

$ sudo sed -i -e 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
$ sudo reboot
$ vagrant reload
$ vagrant ssh
$ getenforce

$ sudo timedatectl set-timezone Asia/Tokyo
$ timedatectl status

$ sudo yum reinstall -y glibc-common
$ localectl list-locales

$ sudo localectl set-locale LANG=ja_JP.utf8
$ sudo localectl set-keymap jp106
$ localectl status
```
```
#日本語フォント インストール（必要に応じて）
$ sudo yum install ibus-kkc vlgothic-*
```

### install rails

```
$ sudo yum install git gcc-c++ openssl-devel readline-devel zlib-devel -y

$ git clone https://github.com/rbenv/rbenv.git ~/.rbenv
$ cd ~/.rbenv && src/configure && make -C src
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$ source ~/.bash_profile
$ rbenv --version

$ mkdir -p "$(rbenv root)"/plugins
$ git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build
$ cd "$(rbenv root)"/plugins/ruby-build && git pull

$ rbenv install -l
$ rbenv install 2.5.1
$ rbenv global 2.5.1
$ rbenv rehash
$ ruby -v
$ rbenv versions
$ curl -fsSL https://github.com/rbenv/rbenv-installer/raw/master/bin/rbenv-doctor | bash

$ gem install bundler

$ gem install rails
$ rails -v
```

### install nginx

```
$ sudo vi /etc/yum.repos.d/nginx.repo
+ [nginx]
+ name=nginx repo
+ baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
+ gpgcheck=0
+ enabled=1

$ sudo yum info nginx
$ sudo yum install nginx -y
$ nginx -v

$ sudo systemctl start nginx
$ sudo systemctl enable nginx
$ sudo systemctl status nginx

http://192.168.33.10/
```

### install postgresql

```
$ sudo vi /etc/yum.repos.d/CentOS-Base.repo
  [base]
  name=CentOS-$releasever - Base
  mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
  #baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
  gpgcheck=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
+ exclude=postgresql*

$ sudo yum localinstall https://download.postgresql.org/pub/repos/yum/testing/11/redhat/rhel-7-x86_64/pgdg-centos11-11-1.noarch.rpm -y
$ sudo yum info postgresql-server
$ sudo yum install postgresql-server -y
$ sudo yum install postgresql-devel -y
$ psql --version

$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb

$ sudo systemctl start postgresql-11
$ sudo systemctl enable postgresql-11
$ sudo systemctl status postgresql-11

$ sudo -u postgres psql -U postgres
postgres=# \q
$ sudo passwd postgres

$ su - postgres
-bash-4.2$ createuser vagrant -s
-bash-4.2$ exit

$ ls -l /usr/pgsql-11/bin/pg*
$ gem install pg -- --with-pg-config=/usr/pgsql-11/bin/pg_config
$ cd /vagrant
$ rails new app-name --force --database=postgresql
```

### start rails
```
$ cd app-name
$ bundle exec spring binstub --all
$ vi Gemfile
- # gem 'mini_racer', platforms: :ruby
+ gem 'mini_racer', platforms: :ruby

$ bundle install
$ rake db:create
$ rails s

http://192.168.33.10:3000/
```

### test migration

```
$ rails generate model Article title:string text:text
$ rake db:migrate:status
```

### connect puma-nginx by unix-socket

```
$ mkdir -p puma_shared/sockets

$ vi config/puma.rb
+ shared_dir = "/puma_shared"
+ # Set up socket location
+ bind "unix://#{shared_dir}/sockets/puma.sock"

$ sudo mv puma_shared /
$ cd /vagrant/app-name
$ bundle exec puma
Ctrl-C
```
```
$ vi app-name.conf
+ upstream app-name {
+     server unix:///puma_shared/sockets/puma.sock;
+ }
+ server {
+     listen       80;
+     server_name  192.168.33.10;
+     location / {
+         proxy_pass http://app-name;
+     }
+ }

$ sudo mv app-name.conf /etc/nginx/conf.d/
$ cd /etc/nginx/conf.d/
$ sudo nginx -t
$ sudo systemctl restart nginx
```
```
$ cd /vagrant/app-name
$ bundle exec puma

http://192.168.33.10/
```

##### reference

- [puma/config\.rb at e0f544d0ec5769c05e439ba251b5685cb546fed5 · puma/puma](https://github.com/puma/puma/blob/e0f544d0ec5769c05e439ba251b5685cb546fed5/examples/config.rb)
- [puma/nginx\.md at master · puma/puma](https://github.com/puma/puma/blob/master/docs/nginx.md)
- [an example of puma and ngxin with ruby on rails config\.md](https://gist.github.com/duleorlovic/762c4ffdf43c8eb31aa7)
- [Rails \+ Puma \+ Nginx on CentOS 7 の 環境構築メモ \- 適当おじさんの適当ブログ](http://www.subarunari.com/entry/2018/04/04/Rails_%2B_Puma_%2B_Nginx_on_CentOS_7_%E3%81%AE_%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89%E3%83%A1%E3%83%A2)
- [RackサーバーのPumaについて調べてみる \- ゆーじのろぐ](http://arakaji.hatenablog.com/entry/2015/08/03/200502)
- [rails5\+puma\+nginxで本番環境を構築してみる – 不動産屋のエンジニア](https://www.gakusmemo.com/?p=608)
