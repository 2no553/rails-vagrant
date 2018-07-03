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
$ git clone https://github.com/2no553/rails-vagrant.git
$ cd rails-vagrant
$ vagrant up
$ vagrant ssh
$ cd /vagrant/app-name
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

#view rails start
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

#view rails start
http://192.168.33.10/
```

##### reference
- [puma/config\.rb at e0f544d0ec5769c05e439ba251b5685cb546fed5 · puma/puma](https://github.com/puma/puma/blob/e0f544d0ec5769c05e439ba251b5685cb546fed5/examples/config.rb)
- [puma/nginx\.md at master · puma/puma](https://github.com/puma/puma/blob/master/docs/nginx.md)
- [an example of puma and ngxin with ruby on rails config\.md](https://gist.github.com/duleorlovic/762c4ffdf43c8eb31aa7)
- [Rails \+ Puma \+ Nginx on CentOS 7 の 環境構築メモ \- 適当おじさんの適当ブログ](http://www.subarunari.com/entry/2018/04/04/Rails_%2B_Puma_%2B_Nginx_on_CentOS_7_%E3%81%AE_%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89%E3%83%A1%E3%83%A2)
- [RackサーバーのPumaについて調べてみる \- ゆーじのろぐ](http://arakaji.hatenablog.com/entry/2015/08/03/200502)
- [rails5\+puma\+nginxで本番環境を構築してみる – 不動産屋のエンジニア](https://www.gakusmemo.com/?p=608)
