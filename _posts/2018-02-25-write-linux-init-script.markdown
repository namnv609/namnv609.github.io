---
date: 2018-02-15 15:38:00 +0700
title: Write Linux Init Script
is_editor_choice: true
---

Như ở bài viết [**Managing services with update-rc.d**](https://viblo.asia/p/managing-services-with-update-rcd-djeZ1x7oKWz) mình đã giới thiệu cách sử dụng **update-rc.d** để cho một service script chạy khi khởi động. Hôm nay mình sẽ giới thiệu cách viết một service script đơn giản nhất (cũng đầy đủ chức năng cơ bản là **start**, **stop**, **restart** và **status**). Chúng ta sẽ viết service cho Unicorn nhé.<!--more-->

## Nguyên liệu

Về phần nguyên liệu, chúng ta sẽ thử nghiệm luôn trên local cho nhanh (đỡ phải động chạm đến server hoặc đỡ phải cài cắm quá nhiều - nếu bạn dùng Docker).

### Source code

Bạn có thể sử dụng ngay [https://github.com/namnv609/test-deploy-ruby-v3](https://github.com/namnv609/test-deploy-ruby-v3) này của mình cho tiện nhé. Clone nó về máy và duplicate file `config/unicorn/staging.rb` thành `config/unicorn/development.rb` rồi sửa lại line 1 chỗ `app_path` cho phù hợp với đường dẫn của project.

Tiếp theo, bạn sửa lại nội dung của file `config/database.yml` ở phần `default:` và `development:` cho phù hợp là xong.

### Nginx

Phần cài đặt này là để chúng ta access vào web thông qua socket của Unicorn với  Nginx. Các bạn thực hiện theo các bước sau:

1. Tạo một file cấu hình cho Nginx trong thư mục `/etc/nginx/sites-available/`, mình lấy tạm tên là `sample_app` và thêm nội dung sau (các bạn nhớ thay placeholder `<Path to project foler>` cho phù hợp):

    ```
    upstream test_deploy_v3 {
      server unix:<Path to project folder>/tmp/sockets/unicorn.sock fail_timeout=0;
    }

    server {
      listen 80;
      server_name sample-app.local;
      root <Path to project folder>/public;

      location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
      }

      location ~ ^/(robots.txt|sitemap.xml.gz)/ {
        root <Path to project folder>/public;
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

2. Enable Nginx config bằng lệnh sau: `sudo ln -s /etc/nginx/sites-available/sample_app /etc/nginx/sites-enabled`
3. Khởi động lại Nginx để áp dụng các cài đặt: `sudo service nginx restart`
4. Thêm sample-app.local vào file `/etc/hosts` để có thể truy cập vào ứng dụng thông qua domain đã cài đặt với Nginx bằng cách mở file `/etc/hosts` và thêm dòng sau:

    ```
    # /etc/hosts
    127.0.0.1   sample-app.local
    ```

Sau khi hoàn thành, bạn có thể thử truy cập vào địa chỉ http://sample-app.local. Nếu kết quả như bên dưới là bạn đã thành công

![](/assets/images/posts/096c8b51-f2bd-4db4-a9a9-6f746f4012ae.png)

Tạm thời bạn chưa cần quan tâm đến lỗi xuất hiện nhé. Vì chúng ta chưa khởi động Unicorn. Vậy là đã xong phần chuẩn bị nguyên liệu, giờ chúng ta bắt tay vào viết service nào.

## Viết script

Bây giờ chúng ta đi vào chi tiết viết một init script đơn giản nhé. Việc đầu tiên chúng ta cần làm là tạo một file có thể thực thi (executable) trong thư mục `/etc/init.d/`, mình lấy tên là `unicorn` cho nó nhanh gọn :D

* `sudo touch /etc/init.d/unicorn && sudo chmod +x /etc/init.d/unicorn`

Tiếp đến, chúng ta thêm đoạn header cho file script với nội dung sau:

```bash
#!/bin/bash
#
# chkconfig:            2345 70 30
# description:          Sample app Unicorn
# processname:          unicorn
#
### BEGIN INIT INFO
# Provides:             unicorn
# Required-Start:       $remote_fs $syslog
# Required-Stop:        $remote_fs $syslog
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    Start Rails app at boot time
# Description:          Enable Rails app running by Unicorn daemon
### END INIT INFO
```

Mình sẽ giải thích về các keywords trên:

* CentOS keywords:
    * `chkconfig`: **2345** là run-levels, **70** là độ ưu tiên khi khởi động, còn **30** là độ ưu tiên khi dừng (các bạn có thể đọc thêm [**tại đây**](https://viblo.asia/p/managing-services-with-update-rcd-djeZ1x7oKWz)). Keyword này để sử dụng cho `chkconfig` của CentOS.
    * `description`: Mô tả cho init script
    * `processname`: Tên init script
* Debian keywords:
    * `Provides`: Chúng ta có thể tạm hiểu là tên khi init script được chạy. Nó phải là duy nhất (unique)
    * `Required-Start`: Yêu cầu phải có các thành phần được yêu cầu trước khi chạy script. Các bạn có thể đọc thêm [**tại đây**](http://refspecs.linuxbase.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/facilname.html)
    * `Required-Stop`: Giống `Required-Start`, nhưng dành cho lúc dừng script
    * `Default-Start`: Là run-levels của script khi chạy. Giống tham số đầu tiên của keyword `chkconfig`
    * `Default-Stop`: Giống `Default-Start`, nhưng dành cho lúc dừng script.
    * `Short-Description`: Mô tả ngắn gọn cho script
    * `Description`: Mô tả đầy đủ cho script

Vậy là đã xong phần header của script. Bây giờ chúng ta tiếp tục viết nhé. Tiếp theo, bạn thêm dòng sau:

```bash
set -e
```

Theo như man page của Ubuntu thì lệnh `set` giúp chúng ta đặt hoặc bỏ chọn giá trị các tùy chọn của shell. Tham  số `-e` có ý nghĩa là thoát ngay lập tức nếu một câu lệnh nào đó trả ra một exit code khác 0. Tiếp, chúng ta đi khai báo một số biến sẽ dùng trong script:

```bash
USER=<Current username>
APP_NAME="Sample App"
APP_ROOT="<Path to project folder>"
PID_FILE="$APP_ROOT/tmp/pids/unicorn.pid"
APP_ENV="development"
CD_CMD="cd $APP_ROOT && (export RAILS_ENV=\"$APP_ENV\";"
BUNDLER_CMD="<Path to RVM> default do bundle exec"
# BUNDLER_CMD="<Path to Bundler> exec"
```

Sau đây là chi tiết các biến được khai báo:

* `USER`: Là user bạn sẽ chạy lệnh. Trên server bạn thường chạy Unicorn ở user khác `root` (như `deploy` chả hạn). Còn đây là chúng ta đang chạy ở local nên bạn thay bằng user hiện tại cho phù hợp. Bạn có thể thực hiện lệnh `whoami` để xem.
* `APP_NAME`: Đơn giản chỉ sử dụng cho phần hiển thị ở phía sau
* `APP_ROOT`: Đường dẫn đến thư mục của project
* `PID_FILE`: Đường dẫn đến file chứa process ID của Unicorn
* `APP_ENV`: Môi trường khi chạy Rails app. Bạn có thể thay đổi nó cho phù hợp
* `CD_CMD`: String chứa lệnh start Unicorn trong thư mục project
* `BUNDLER_CMD`: Thực hiện chạy Unicorn bằng `bundler`. Bạn có thể sử dụng một trong hai và sửa lại đường dẫn cho phù hợp. Nếu bạn dùng RVM thì nên dùng cái đầu tiên. Bạn có thể chạy lệnh `which rvm` hoặc `which bundle` để lấy đường dẫn đến file thực thi của RVM hoặc Bundler.

Tiếp, chúng ta đi viết hai function là lấy PID hiện tại của Unicorn và kiểm tra xem nó có chạy không để sử dụng cho các function như **start**, **stop**, **restart**, **status**, **reload**:

```bash
get_pid() {
  cat "$PID_FILE"
}

is_running() {
  [ -f "$PID_FILE" ] && kill -0 $(get_pid) > /dev/null 2>&1
}
```

Trong hai function trên. Function `get_pid()` thì không có gì đặc biệt ngoài việc đọc file PID. Còn function `is_running()` thì có một lệnh khá lạ là `kill -0`. Với signal 0 có nghĩa là lệnh kill này không gửi signal tới process đó nhưng nó vẫn kiểm tra xem có lỗi khi kill hay không. Chúng ta sử dụng nó để kiểm tra xem Unicorn có đang chạy hay không. Còn đoạn ` > /dev/null 2>&1` đơn giản chỉ là không hiển thị `stdout` và `stderr` (trong trường hợp PID đó không tồn tại). Đưa tất cả về lỗ đen `/dev/null` :D. Okay, tiếp theo chúng ta đi viết script xử lý các tham số nhận vào khi chạy service nhé. Tại sao lại viết từ đoạn này? Là mình muốn để mọi người viết function nào thì có thể test luôn function đó. Đoạn code này nó phải ở cuối. Dưới các function sẽ được khai báo tiếp theo nhé:

```bash
case "$1" in
  start)
    do_start
    ;;
  stop)
    do_stop
    ;;
  restart)
    do_restart
    ;;
  status)
    do_status
    ;;
  reload)
    do_reload
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|reload|status}"
    ;;
esac
```

Trong khối lệnh `case` ở trên không có gì đặc biệt ngoài đọc tham số đầu tiên (là `$1`) rồi gọi function tương ứng. Nếu `$1` không nằm trong các case được định nghĩa thì in thông báo hướng dẫn sử dụng. Tiếp, chúng ta đi viết function kiểm tra trạng thái của Unicorn có đang chạy hay không:

```bash
do_status() {
  if is_running; then
    echo "$APP_NAME (process $(get_pid)) is running..."
  else
    echo "$APP_NAME is stopped"
  fi
}
```

Trong function trên không có gì đặc biệt cần phải giải thích. Bây giờ bạn thử thực thi lệnh `sudo service unicorn status` hoặc `sudo service unicorn` để xem kết quả ra sao :D! Giờ chúng ta đi viết lần lượt các function còn lại nhé. Đầu tiên là `do_start`.

```bash
do_start() {
  if is_running; then
    echo "$APP_NAME already started"
  else
    echo "Starting $APP_NAME..."
    su - $USER -c "$CD_CMD $BUNDLER_CMD unicorn -c $APP_ROOT/config/unicorn/$APP_ENV.rb -E $APP_ENV -D)"
    sleep 5
    echo "$APP_NAME started with process $(get_pid)"
  fi
}
```

Cũng không có gì đặc biệt ở function `do_start()` này. Đơn giản chỉ là kiểm tra nếu Unicorn đang chạy thì hiển thị thông báo. Còn không thì start Unicorn và đợi 5s (thời gian tương đối để đợi Unicorn được khởi động - đôi khi cũng hên xui :D). Giờ bạn có thể thử hai lệnh start và status để xem kết quả:

```
$ sudo service unicorn status
Sample App is stopped
$ sudo service unicorn start
Starting Sample App...
Sample App started with process 10500
$ sudo service unicorn status
Sample App (process 10500) is running...
$ ps ax | grep "[u]nicorn master"
10500 ?        Sl     0:01 unicorn master -c <Path to project folder>/config/unicorn/development.rb -E development -D
```

Đã có start, bây giờ chúng ta sang phần stop:

```bash
do_stop() {
  if is_running; then
    echo "Stopping $APP_NAME..."
    su - $USER -c "/usr/bin/env kill -s QUIT $(get_pid)"
    sleep 5
    if is_running; then
      echo "$APP_NAME not stopped. May still be shutting down or shutdown may have failed"
      exit 1
    else
      echo "$APP_NAME stopped"
    fi
  else
    echo "$APP_NAME not running"
  fi
}
```

Ở function `do_stop()` này. Chúng ta sẽ kiểm tra xem, nếu Unicorn đang chạy thì thực hiện kill nó với signal QUIT và đợi 5s (cũng vẫn là con số tương đối :rofl:) rồi kiểm tra lại xem nó đã thực sự được stop hay chưa. Bây giờ bạn có thể thử lệnh stop (sau khi đã thử lệnh start ở trên) rồi tiếp lệnh status xem sao :smile:

```
$ sudo service unicorn status
Sample App (process 10939) is running...
$ sudo service unicorn stop
Stopping Sample App...
Sample App stopped
$ sudo service unicorn status
Sample App is stopped
```

Xong phần stop. Chúng ta sang phần restart. Phần này là đơn giản nhất. Chỉ cần kiểm tra xem Unicorn có chạy hay không. Nếu đang chạy rồi thì stop nó và start lại. Nếu chưa chạy thì start nó lên:

```bash
do_restart() {
  if ! is_running; then
    echo "$APP_NAME not running."
    do_start
  else
    do_stop
    do_start
  fi
}
```

Bây giờ, bạn thử thực hiện việc restart xem nó đã chạy đúng chưa. Dưới đây là test case của mình:

```
$ sudo service unicorn status
Sample App is stopped
$ sudo service unicorn restart
Sample App not running.
Starting Sample App...
Sample App started with process 11262
$ sudo service unicorn status
Sample App (process 11262) is running...
$ sudo service unicorn restart
Stopping Sample App...
Sample App stopped
Starting Sample App...
Sample App started with process 11458
```

Vậy là xong phần restart. Với ứng dụng khác thì có thể là đã đủ, nhưng với Unicorn thì vẫn chưa đủ. Vì Unicorn nó có hỗ trợ gửi signal reload lại process ([Unicorn signal handling](https://bogomips.org/unicorn/SIGNALS.html)) để nhận code mới mà không gây gián đoạn đến việc truy cập vào trang của người dùng (bạn có thể xem thêm bài [**Zero downtime deployment for Rails with Capistrano and Unicorn**](/posts/zero-downtime-deployment-for-rails-with-capistrano-and-unicorn.html) để hiểu hơn) nên chúng ta thêm lệnh reload (nếu bạn thấy cần) nhé. Function `do_reload()` này đơn giản chỉ là kiểm tra xem Unicorn có đang chạy hay không. Nếu đang chạy thì gửi signal `USR2` tới master PID là xong.

```bash
do_reload() {
  if ! is_running; then
    echo "$APP_NAME not running."
  else
    echo "Reloading $APP_NAME..."
    su - $USER -c "/usr/bin/env kill -s USR2 $(get_pid)"
    sleep 5
    echo "$APP_NAME reloaded"
  fi
}
```

Xong phần reload. Bạn có thể kiểm tra theo test case sau của mình:

```
$ sudo service unicorn status
Sample App (process 11742) is running...
$ ps ax | grep "[u]nicorn master"
11742 ?        Sl     0:01 unicorn master -c <Path to project folder>/config/unicorn/development.rb -E development -D
$ sudo service unicorn reload
Reloading Sample App...
Sample App reloaded
$ sudo service unicorn status
Sample App (process 11872) is running...
$ ps ax | grep "[u]nicorn master"
11872 ?        Sl     0:01 unicorn master -c <Path to project folder>/config/unicorn/development.rb -E development -D
```

Misson completed :smiley:!

# Lời kết

Đến đây, bài viết chia sẻ làm sao để viết được một Linux init script đơn giản đã kết thúc. Hy vọng hai bài viết là bài này và bài [**Managing services with update-rc.d**](https://viblo.asia/p/managing-services-with-update-rcd-djeZ1x7oKWz) sẽ giúp ích cho mọi người trong một thời điểm nào đó. Service này mình viết vẫn còn non tay (do phải mày mò và tổng hợp rất nhiều cách viết mà mình đã tìm kiếm trên mạng) nên có gì thiếu sót mong mọi người thông cảm và góp ý ở phần bình luận nhé. Chào thân ái và quyết thắng :wave:!
