---
date: 2017-08-26 16:11:00 +0700
title: Zero downtime deployment for Rails with Capistrano and Unicorn
is_editor_choice: true
---

Trên Viblo cũng có khá nhiều bài viết về việc auto deploy một ứng dụng Ruby on Rails với Capistrano. Nhưng mình cũng vẫn chia sẻ bài viết này với mục đích hướng dẫn mọi người chi tiết hơn trong việc cài đặt một server từ chưa có gì cho tới khi ứng dụng của chúng ta được chạy và có khả năng deploy hoàn chỉnh.<!--more--> Nằm ngoài mục đích đó là mình cũng muốn note lại và tổng kết những gì mình đã tìm hiểu, học được từ việc cài đặt và sử dụng Capistrano. Chúng ta cùng vừa đọc vừa thực hành luôn nhé. Việc khó khăn nhất là chúng ta cần phải có một server để thực hiện, nhưng may mắn thay, chúng ta có thể sử dụng Docker để giả lập một server. Về việc cài đặt và sử dụng Docker như thế nào thì mình không bàn tới trong bài viết này nhé. Mặc định là mọi người đã cài và có thể sử dụng Docker căn bản. Chúng ta cùng đi từng bước một nhé. Phần đầu là chúng ta thực hiện cài đặt một server có thể SSH vào được từ host (giống với môi trường server thật).

## Cài đặt server

Việc sử dụng image nào là tùy mọi người. Mình sử dụng image Ubuntu 14.04 nhé. Pull image (nếu chưa có):

```
sudo docker pull ubuntu:14.04
```

Sau khi pull image xong. Bạn có thể kiểm tra danh sách các images bằng lệnh:

```
sudo docker images
```

Khi đã có image của Ubuntu, chúng ta khởi tạo một Docker container bằng lệnh:

```
sudo docker run --name virtual_server -it ubuntu:14.04 /bin/bash
```

> Sau khi khởi tạo xong, bạn đã ở trong container, mặc định chúng ta đang ở quyền cao nhất rồi (root) nên các câu lệnh không cần prefix là `sudo` nữa nhé (nhưng ở môi trường thật nếu chúng ta chưa ở root thì vẫn cần phải có prefix này để thực hiện các câu lệnh liên quan đến hệ thống yêu cầu quyền super user).

Tiếp theo, chúng ta sẽ cài đặt một package giúp chúng ta có thể SSH vào container là **OpenSSH server**. Đầu tiên, update lại APT cache bằng lệnh:

```
apt-get update
```

Sau đó, cài OpenSSH server và VIM để cài thực hiện cài đặt nhé:

```
apt-get install -y vim openssh-server
```

Sau khi cài đặt xong, chúng ta sang phần cấu hình OpenSSH server để có thể SSH từ host vào container nhé. Trước khi sửa file cấu hình của OpenSSH, chúng ta cứ backup lại một bản cho chắc:

```
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Thực hiện sửa file `sshd_config` bằng lệnh `vim /etc/ssh/sshd_config` và sửa các cài đặt sang các giá trị sau:

> Note: Trong VIM, bạn có thể search bằng cách dùng `/<pattern|keyword|string>` để search. Ví dụ: `/MaxAuthTries`

```
MaxAuthTries 3 # Bạn có thể thay đổi sang số khác, nếu thích
RSAAuthentication yes
PubkeyAuthentication yes
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
```

Thoát VIM và khởi động lại SSH server bằng lệnh:

```
service ssh restart
```

Tạo thư mục `.ssh`  để chứa publickey của máy mà chúng ta có thể SSH lên:

```
ssh-keygen -t rsa
# Sau đó bạn cứ enter cho đến hết (nếu muốn :D)
```

Vào thư mục `~/.ssh` và tạo một file với tên `authorized_keys` để chứa publickey với lệnh:

```
touch authorized_keys && chmod 600 authorized_keys
```

Add publickey của host vào `authorized_keys` của container bằng cách bước sau:
* Host:
    * `cat ~/.ssh/id_rsa.pub` và copy đoạn SSH key
* Container
    * `vi ~/.ssh/authorized_keys` và paste đoạn SSH của host vào

Chúng ta thử SSH từ host vào container nhé. Để xem IP của container, chúng ta có thể sử dụng `ifconfig` trong container hoặc lệnh sau trên host:

```
sudo docker inspect virtual_server | grep IPAddress
```

Của mình là `172.17.0.2` (còn của bạn bạn có thể sẽ khác, nếu có nhiều container đã được khởi tạo trước đó). Thử SSH vào xem sao:

```
ssh root@172.17.0.2
```

OK, thế là xong phần basic. Bây giờ chúng ta đi cài đặt các thành phần cần thiết để có thể chạy được một ứng dụng Rails là Ruby, MySQL, NginX, ... nhé. Chúng ta thực hiện cài đặt các packages cần thiết như khi làm việc với một server thật là không dùng tới tab access vào docker container trước mà chúng ta sử dụng tab mới vừa SSH ở trên để thực hiện nhé :D!

> Do chúng ta tạo container từ image nên bạn luôn phải giữ cho Docker container được up. Mọi công việc khác trên host nên làm ở tab terminal khác và không được terminate tab Docker hiện tại.

### Cài đặt Ruby

Chúng ta sử dụng RVM cho tiện nhé. Cài đặt CURL (nếu chưa có - chắc chắn là image 14.04 này chưa có package CURL đâu :D):

```
apt-get install -y curl
```

Cài đặt RVM (theo trang chủ của [RVM.io](https://rvm.io)):

```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

curl -sSL https://get.rvm.io | bash -s stable
```

Sau khi cài đặt xong, chúng ta thực hiện lệnh: `source /usr/local/rvm/scripts/rvm` để có thể dùng luôn RVM trên terminal hiện tại. Ngoài ra, chúng ta nên thêm đoạn đó vào file `~/.bashrc`, `~/.profile` hoặc `~/.bash_profile` để lần sau chúng ta khởi động và access vào container là có thể sử dụng luôn RVM bằng lệnh:

```
echo source /usr/local/rvm/scripts/rvm | tee -a ~/.bashrc
```

Sau khi cài đặt xong RVM, chúng ta thực hiện lệnh `rvm requirements` để RVM kiểm tra và cài đặt các packages cần thiết để có thể cài đặt Ruby.
Sau khi RVM cài đặt xong các packages cần thiết, chúng ta cài Ruby (mình cài bản 2.3.1) bằng lệnh:
```
rvm install ruby-2.3.1
```

Tiếp theo, chúng ta cài `Bundler` và `Rails` bằng lệnh:
```
gem install bundler
gem install rails
```

Đến đây là chúng ta xong phần cài đặt Ruby, giờ chúng ta chuyển sang cài đặt các thành phần khác như NginX, MySQL nhé

### Cài đặt MySQL

Để cài đặt MySQL, ở trên môi trường server có lẽ bạn không cần phải đi qua các bước như với Docker image 14.04 này. Mình sử dụng MySQL 5.6 nhé (nhưng do image này chỉ có bản 5.5).

```
apt-get install -y software-properties-common python-software-properties libmysqlclient-dev
add-apt-repository 'deb http://archive.ubuntu.com/ubuntu trusty universe'
apt-get update
apt-get install mysql-server-5.6
```

Sau khi cài đặt xong MySQL, chúng ta start MySQL bằng lệnh: `service mysql start` hoặc `/etc/init.d/mysql start`. Sau khi bật MySQL service, chúng ta thử xem mọi thứ có hoạt động không bằng lệnh sau:

```
mysql -uroot -p -e "show databases;"
```

Sau đó bạn nhập password khi cài đặt. Chúng ta tiếp tục sang phần cài đặt NginX, Git, Node.JS và NPM nhé :D!

### Cài đặt NginX, Git, Node.JS và NPM

#### Cài đặt Git

Chúng ta đi cài đặt Git để có thể pull source code khi deploy nhé.

```
apt-get install git
```

Sau khi cài xong, bạn có thể kiểm tra version của git bằng lệnh `git --vesion`

#### Cài đặt Node.JS và NPM

Node.JS và NPM để dùng khi precompile assets trong Rails.

```
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
```

Sau đó, cài Node.JS (bao gồm cả NPM) bằng lệnh:

```
apt-get install nodejs
```

Sau khi cài đặt xong, bạn thử một trong hai lệnh sau để kiểm tra: `nodejs -v` hoặc `node -v`. Nếu lệnh `node -v` không ra kết quả, bạn có thể dùng lệnh sau để tạo symlink (vì có một số package dùng `node` chứ không dùng `nodejs`):

```
ln -s `which nodejs` /usr/local/bin/node
```

Tiếp theo, chúng ta sẽ cài NginX để chạy ứng dụng Rails của chúng ta nhé

#### Cài đặt NginX

Thêm repository của NginX:

```
add-apt-repository ppa:nginx/stable
```

Sau đó, update lại APT cache:

```
apt-get update
```

Cài đặt NginX:

```
apt-get install nginx
```

Sau khi cài đặt xong, bật NginX bằng lệnh: `service nginx start` và bạn có thể kiểm tra bằng lệnh: `curl localhost` hoặc dùng browser truy cập vào IP của Docker container.

OK, đã xong phần cài đặt server. Giờ chúng ta sang phần cài đặt trong source code để có thể auto deploy nhé.

> Do chúng ta cài đặt container từ image nên khi bạn start lại Docker container, bạn cần vào container bật các dịch vụ như NginX, MySQL, SSH. Các services này không startup cùng container khi nó được start
> Các service cần bật:
> ```
> service ssh start
> /etc/init.d/mysql start
> service nginx start
> ```

## Cài đặt source code

Bạn có thể áp dụng với một project có sẵn (hoặc có thể tạo mới để kiểm thử). Chúng ta cùng đi vào các bước chi tiết nhé. Đầu tiên, bạn thêm các gem sau vào file `Gemfile` của project:

```ruby
gem "capistrano"
gem "capistrano-rails"
gem "capistrano3-unicorn"
gem "unicorn"
gem "capistrano-rvm"
```

Sau đó, chạy lệnh `bundle install` để cài đặt các gem mới. Sau khi đã thêm các gem cần thiết xong, chúng ta sang phần cài đặt cho Capistrano nhé. Để khởi tạo file cài đặt cho Capistrano, bạn thực hiện lệnh dưới đây:

```
bundle exec cap install
```

Sau khi Capistrano đã tạo ra các file và thư mục cần thiết, bạn hãy mở file `Capfile` và thực hiện các cài đặt sau:

```ruby
# Uncomment các require sau
require "capistrano/rvm"
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
# Thêm require sau
require "capistrano3/unicorn"
```

Xong, bạn có thể thử xem đã có tasks của Unicorn trong Capistrano chưa bằng lệnh: `bundle exec cap -T | grep unicorn`. Tiếp theo, chúng ta mở file `config/deploy.rb` để cài đặt một chút nhé.

```ruby
set :application, "test_deploy_v3"
set :repo_url, "git@github.com:namnv609/test-deploy-ruby-v3.git"
set :bundle_binstubs, nil

# Default deploy_to directory is /var/www/my_app_name
set :deploy_to, "/var/www/html/#{fetch(:application)}"

# Default value for :linked_files is []
set :linked_files, fetch(:linked_files, [])
  .push("config/database.yml", "config/secrets.yml")
# Default value for linked_dirs is []
set :linked_dirs, fetch(:linked_dirs, [])
  .push("log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "vendor/bundle")

# Default value for keep_releases is 5
set :keep_releases, 5

after "deploy:publishing", "deploy:restart"

# Khởi động lại unicorn sau khi deploy
namespace :deploy do
  task :restart do
    invoke "unicorn:restart"
  end
end
```

Mình giải thích một chút nhé:

* `:application` là tên ứng dụng sẽ deploy
* `:repo_url` là Github repository URL
* `:deploy_to` là thư mục sẽ chứa code deploy
* `:linked_files` là các file dùng chung cho các bản deploy như `secrets.yml`, `.env`, `database.yml`, ...
* `:linked_dirs` là các thư mục dùng chung cho các bản deploy
* `:keep_releases` là số lượng bản deploy sẽ giữ lại. Tương đương với số lần bạn có thể rollback lại

Sau đó, bạn mở file `config/deploy/staging.rb` (bạn có thể sửa file `production.rb` hoặc có thể tạo các stage khác ví dụ như `aws_staging.rb`, `testing.rb`, ... nếu bạn muốn, trong ví dụ này thì mình dùng file `staging.rb`) và thêm/sửa các dòng sau:

```ruby
set :user, "root"
set :deploy_via, :remote_cache
set :conditionally_migrate, true
set :rails_env, "staging"

# Phần IP thì bạn thay thế cho phù hợp với IP của Docker container nhé
server "172.17.0.2", user: fetch(:user), port: fetch(:port), roles: %w(web app db)
```

Trong phần trên, mình giải thích sơ qua một chút. `:user` là user sẽ sử dụng để deploy trên server (trong ví dụ này là `root`, ngoài thực tế có thể là user khác), `:roles` các các roles mà Capistrano sẽ sử dụng, nếu bạn có nhiều hơn một server, thì chỉ cần một server có chứa role db thôi (vì nhiều server cũng chỉ chung nhau một database và chỉ cần chạy migrate trên một server là đủ), ví dụ:

```ruby
server "172.17.0.2", user: fetch(:user), port: fetch(:port), roles: %w(web app db)
server "172.17.0.3", user: fetch(:user), port: fetch(:port), roles: %w(web app)
```

Sau khi xong, bạn thử xem Capistrano có thể tương tác với server được hay chưa bằng lệnh:

```
bundle exec cap staging deploy:check
```

Sẽ có một số lỗi liên quan đến việc Capistrano không tìm thấy các file được khai báo trong `:linked_files`, bạn cần phải vào server và tạo các file đó, ở đây là chúng ta sẽ tạo hai files là `secrets.yml` và `database.yml` trong thư mục `/var/www/html/#{fetch(:application)}/shared/config/` với nội dung là các cài đặt riêng cho môi trường staging. Sau khi xong, bạn có thể thử lại lệnh trên và xem kết quả :D!

Tiếp, chúng ta sẽ tạo thư mục `unicorn` trong thư mục `config` để chứa các cài đặt cho Unicorn theo từng stage nhé. Các cài đặt này giúp cho việc sẽ không bị delay ảnh hưởng đến người dùng trong quá trình deploy. Ở đây, chúng ta sẽ tạo file `config/unicorn/staging.rb` và thêm vào nội dung sau:

```ruby
app_path = "/var/www/html/test_deploy_v3/current"
working_directory app_path

pid "#{app_path}/tmp/pids/unicorn.pid"

stderr_path "#{app_path}/log/unicorn.err.log"
stdout_path "#{app_path}/log/unicorn.out.log"

worker_processes 3
timeout 30
preload_app true

listen "#{app_path}/tmp/sockets/unicorn.sock", backlog: 64

before_exec do |_|
  ENV["BUNDLE_GEMFILE"] = File.join(app_path, "Gemfile")
end

before_fork do |server, worker|
  defined?(ActiveRecord::Base) and ActiveRecord::Base.connection.disconnect!

  old_pid = "#{app_path}/tmp/pids/unicorn.pid.oldbin"

  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end
end

after_fork do |server, worker|
  defined?(ActiveRecord::Base) and ActiveRecord::Base.establish_connection
end
```

Trong file trên, bạn chỉ cần quan tâm đến `app_path` và sửa sao cho đúng với đường dẫn trên server là được nhé.

Tiếp theo, chúng ta tạo một file trong thư mục `/etc/init.d/` trên server để có thể start/restart/reload/stop ứng dụng Rails của chúng ta từ terminal nhé. Tạo file với tên tùy chọn (trong trường hợp này mình đặt là `unicorn_deploy`):

```
vi /etc/init.d/unicorn_deploy
```

Sau đó, thêm đoạn sau:

```sh
#!/bin/sh
set -u
set -e
# Example init script, this can be used with nginx, too,
# since nginx and unicorn accept the same signals
#[[ -s '/usr/local/rvm/scripts/rvm' ]] && source '/usr/local/rvm/scripts/rvm'

# Feel free to change any of the following variables for your app:
USER=root
GEM_HOME="/var/www/html/test_deploy_v3/shared/bundle"
APP_ROOT="/var/www/html/test_deploy_v3/current"
SET_PATH="export GEM_HOME=$GEM_HOME"

PID="$APP_ROOT/tmp/pids/unicorn.pid"
ENV="staging"
CMD="$SET_PATH; cd $APP_ROOT && bundle exec unicorn -D -E $ENV -c $APP_ROOT/config/unicorn/$ENV.rb"
old_pid="$PID.oldbin"

#cd $APP_ROOT || exit 1
$SET_PATH || exit 1

sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
  test -s $old_pid && kill -$1 `cat $old_pid`
}

case $1 in
start)
  sig 0 && echo >&2 "Already running" && exit 0
  su - $USER -c "$CMD"
  ;;
stop)
  sig QUIT && exit 0
  echo >&2 "Not running"
  ;;
force-stop)
  sig TERM && exit 0
  echo >&2 "Not running"
  ;;
restart|reload)
  sig HUP && echo reloaded OK && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  su - $USER -c "$CMD"
  ;;
upgrade)
  sig USR2 && echo upgraded OK && exit 0
  echo >&2 "Couldn't upgrade, starting '$CMD' instead"
  su - $USER -c "$CMD"
  ;;
rotate)
  sig USR1 && echo rotated logs OK && exit 0
  echo >&2 "Couldn't rotate logs" && exit 1
  ;;
*)
  echo >&2 "Usage: $0 <start|stop|restart|upgrade|rotate|force-stop>"
  exit 1
  ;;
esac
```

Ở đoạn trên, bạn cần quan tâm các biến sau:

* `USER` là user sẽ sử dụng để chạy ứng dụng, giống với user trong `config/deploy/<stage>.rb`
* `GEM_HOME` là thư mục chứa Gem của ứng dụng, bạn sửa lại đường dẫn cho phù hợp
* `APP_ROOT` là thư mục chứ ứng dụng, bạn sửa lại cho phù hợp
* `ENV` là môi trường của ứng dụng, bạn sửa lại cho phù hợp

Sau khi lưu lại, bạn cần cấp quyền thực thi (executable) cho file đó với lệnh:

```
chmod +x /etc/init.d/unicorn_deploy
```

Vậy là xong, từ giờ, bạn có thể start/restart/reload/stop ứng dụng Rails từ terminal với lệnh: `/etc/init.d/unicorn_deploy <start|stop|reload|restart>` rồi :D! Chúng ta sang phần cài đặt NginX trên server nhé. Đầu tiên, bạn vào thư mục `/etc/nginx/sites-available/` và backup lại file `default` trước khi sửa đổi:

```
cp default default.bak
```

Sau đó, sửa file `default` với nội dung sau:

```
upstream test_deploy_v3 {
  server unix:/var/www/html/test_deploy_v3/current/tmp/sockets/unicorn.sock fail_timeout=0;
}

server {
  listen 80 default deferred;
  # server_name example.com;
  root /var/www/html/test_deploy_v3/current/public;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location ~ ^/(robots.txt|sitemap.xml.gz)/ {
    root /var/www/html/test_deploy_v3/current/public;
  }

  try_files $uri/index.html $uri @test_deploy_v3;
  location @test_deploy_v3 {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://test_deploy_v3;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
}
```

Bạn cần phải sửa lại các đường dẫn đến thư mục chứa project sao cho phù hợp nhé. Lưu file lại và khởi động lại NginX bằng lệnh:

```
service nginx restart
```

OK, bây giờ bạn push source code lên Github và thử bắt đầu deploy nhé. Chạy lệnh sau trong thư mục project trên host:

```
bundle exec cap staging deploy
```

Sau khi deploy xong, bạn có thể truy cập vào IP của Docker container bằng trình duyệt để kiểm tra xem ứng dụng của chúng ta đã deploy thành công hay chưa.

> Nếu bạn có sửa source code (thay đổi config, ...) trực tiếp trên server thì bạn cần phải khởi động lại Unicorn để những thay đổi đó có tác dụng. Bạn có thể dùng lệnh sau:
> ```
> kill -USR2 `cat <deploy_to path>/shared/tmp/pids/unicorn.pid`
> ```

Đến đây, bài viết giới thiệu về auto deployment cho Rails đã kết thúc. Các bạn có thắc mắc hay gặp vấn đề gì thì để lại comment, chúng ta cùng bàn luận nhé :D! Hẹn gặp lại mọi người trong các bài viết tiếp theo. Thân!

Link Github project: [https://github.com/namnv609/test-deploy-ruby-v3](https://github.com/namnv609/test-deploy-ruby-v3)
