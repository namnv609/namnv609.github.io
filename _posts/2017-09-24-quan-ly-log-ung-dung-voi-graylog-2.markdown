---
date: 2017-09-24 17:58:00 +0700
title: Quản lý log ứng dụng với GrayLog 2
is_editor_choice: true
---

Dự án trước mình tham gia có sử dụng dịch vụ Amazon CloudWatch để quản lý log của ứng dụng. Mình thấy thực sự rất hay ho. Hay ho vì việc hiển thị rất trực quan, dễ dàng cho việc tìm kiếm, thao tác và xem log rất đơn giản thay vì phải SSH vào từng server và xem bằng **Tail** hoặc **Less**, ...<!--more--> Nhưng nó lại thu phí nên mình đã thử mò mẫm trên một số trang và tìm thấy được thằng GrayLog này. Nó cũng na ná giống như Amazon CloudWatch nhưng nó open source. Hôm nay mình sẽ giới thiệu về nó bằng một bài viết hướng dẫn cài đặt GrayLog với ứng dụng của chúng ta để thực hiện quản lý log nhé.

Việc đầu tiên là chúng ta cần một con server để chứa thằng GrayLog này. Với việc demo thì Docker là một sự lựa chọn hoàn hảo (yaoming)! Chúng ta bắt tay vào cài đặt nó trên Docker nhé (mặc định là mọi người đã cài đặt và biết sử dụng căn bản thằng Docker rồi nhé).

Các packages cần thiết:

* Java
* MongoDB
* ElasticSearch
* GrayLog 2

# Cài đặt GrayLog 2

Khởi tạo một container. Mình sử dụng Ubuntu 14.04 nhé

```
sudo docker run --name graylog2_server -it ubuntu:14.04 /bin/bash
```

Bây giờ chúng ta đang ở trong Docker container rồi và mặc định nó ở tài khoản root nên chúng ta thực hiện các lệnh trên terminal mà không cần prefix `sudo`. Thực hiện update APT cache để chuẩn bị cài đặt một số package cần thiết cho quá trình cài đặt GrayLog

```
apt-get update
```

Tiếp tục, chúng ta cài đặt các package cơ bản (do image này là bản minimal nên nó loại bỏ khá nhiều các package cho nhẹ)

```
apt-get install -y curl wget vim software-properties-common python-software-properties
```

Tiếp đến, cài đặt các package cơ bản và cần thiết cho GrayLog

```
apt-get install -y apt-transport-https uuid-runtime pwgen
```

Cài đặt Java

```
add-apt-repository ppa:webupd8team/java -y
apt-get update
apt-get install -y oracle-java8-installer
apt-get install -y oracle-java8-set-default
```

Cài đặt MongoDB để chứa log. Do trong Docker container không hỗ trợ upstart script nên mình cài đặt mặc định luôn từ APT (v2.4.9). Còn ở môi trường thật, bạn có thể cài bản mới nhất thông qua hướng dẫn từ MongoDB nhé.

```
apt-get install -y mongodb-server
```

Sau khi cài đặt xong, khởi động MongoDB

```
service mongodb start
```

Cài đặt ElasticSearch để thằng GrayLog đánh index dữ liệu log

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-5.x.list
apt-get update
apt-get install -y elasticsearch
```

Bỏ comment và sửa `cluster.name` từ `my-application` thành `graylog` bằng lệnh

```
vi /etc/elasticsearch/elasticsearch.yml
```

Lưu lại, thoát VIM và khởi động ElasticSearch

```
service elasticsearch start
```

Nếu muốn, bạn có thể kiểm tra xem ElasticSearch đã hoạt động chưa, bạn có thể thử lệnh

```
curl localhost:9200
```

Nếu OK, kết quả bạn nhận được sẽ có dạng sau:

```json
{
  "name" : "1MiXgwQ",
  "cluster_name" : "graylog",
  "cluster_uuid" : "9tT3cMMjSLqIWyNM3jDSRQ",
  "version" : {
    "number" : "5.6.1",
    "build_hash" : "667b497",
    "build_date" : "2017-09-14T19:22:05.189Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
```

OK, đã xong các thành phần cần thiết. Giờ chúng ta sẽ đi cài đặt thành phần chính là thằng GrayLog

```
wget https://packages.graylog2.org/repo/packages/graylog-2.3-repository_latest.deb
dpkg -i graylog-2.3-repository_latest.deb
apt-get update
apt-get install graylog-server
```

Sau khi cài đặt xong, chúng ta cần phải tạo 2 password để cài đặt cho GrayLog. Một cái là `password_secret` và một cái là `root_password_sha2` (dùng để đăng nhập vào trang quản trị)

```
# Password secret
pwgen -N 1 -s 96

# Root password SHA256
echo -n <yourpassword> | sha256sum
```

> **Chú ý**: Ở lệnh sinh root password sẽ có thêm đoạn `-` sau mật khẩu đã được mã hóa, bạn không cần phải copy đoạn đó. Ví dụ, mình tạo password **123456**
> `echo -n 123456 | sha256sum`
> Chúng ta sẽ nhận được dữ liệu như sau: `8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92  -`
> Bạn chỉ cần `8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92` là đủ

Bạn sửa file `/etc/graylog/server/server.conf` với các giá trị sau

```
password_secret = <Your password secret hash>
root_password_sha2 = <Your password SHA256 hash>
rest_listen_uri = http://0.0.0.0:9000/api/
web_listen_uri = http://0.0.0.0:9000/
root_timezone = Asia/Ho_Chi_Minh
```

> Mục đích sửa `rest_listen_uri` và `web_listen_uri` từ `127.0.0.1` sang `0.0.0.0` là để chúng ta có thể truy cập vào GrayLog thông qua IP của Docker

Lưu lại các thay đổi và khởi động GrayLog server

```
service graylog-server start
```

Giờ đây, bạn có thể truy cập vào GrayLog Web Interface bằng địa chỉ IP của Docker container với URL: `http://<Docker container IP>:9000`. Username là `admin` và password là password mà bạn đã tạo trước đó.

Vậy là đã xong phần cài đặt GrayLog server. Bây giờ chúng ta sang phần cài đặt cho ứng dụng của mình có thể gửi log lên thằng GrayLog này nhé. Mình sẽ sử dụng ứng dụng Ruby on Rails để thử nghiệm. Bạn có thể tạo mới một project hoặc dùng luôn project đang có cũng được nhé :D!

# Sending in log data

## Ruby on Rails

Đầu tiên, bạn truy cập và đăng nhập vào GrayLog Web Interface thông qua IP của Docker container ở port 9000 với tài khoản `admin`. Tại menu **System** bạn chọn **Inputs**. Trong giao diện **Inputs**, tại dropdown list _Select input_ bạn chọn **GELF TCP** và click **Launch new input** rồi điền các thông tin sau rồi bấm **Save**:

* Node: Chọn một node
* Title: Tên của input
* Bind address: 0.0.0.0
* Port: Để mặc định (là 12201)

![GrayLog Launch new input](/assets/images/posts/88934d35-de12-4a58-8cf9-28637da2f0ac.jpeg)

Sau khi tạo xong, tại màn hình **Inputs**, bạn click vào button **Show received messages** tại input vừa tạo và chuyển sang bước cấu hình trong code. Tiếp theo, bạn thêm hai gem sau vào `Gemfile` và chạy `bundle install` để cài đặt

```ruby
gem "gelf"
gem "lograge"
```

Sau khi cài đặt các gem xong, chúng ta mở file `config/environments/development.rb` (chúng ta đang test trên local. Nếu bạn muốn, có thể sửa các môi trường khác) ra và thêm vào các dòng sau:

```ruby
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Graylog2.new
  config.logger = GELF::Logger.new "<Docker container IP>", 12201, "WAN", {protocol: GELF::Protocol::TCP, facility: "Application Name"}
```

Xong, bây giờ bạn thực hiện lệnh `rails serve` và truy cập vào `localhost:3000` để sinh log rồi sang giao diện của GrayLog và Refresh lại trang rồi xem kết quả nhé :D!

<iframe width="560" height="315" src="https://www.youtube.com/embed/QYuT6Q8mDr8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Laravel 5

Bạn thực hiện tạo thêm một input mới giống với phần **Ruby on Rails** với các thông tin sau:

* Node: Chọn một node
* Title: Tên của input
* Bind address: 0.0.0.0
* Port: 12202 (để tránh trùng với port của Rails input)

Tiếp theo, bạn cài đặt package `graylog2/gelf-php` bằng lệnh `composer require graylog2/gelf-php`. Sau khi cài đặt xong, bạn mở file `bootstrap/app.php` và thêm vào các dòng sau:

```php
$app->configureMonologUsing(function($monolog) {
    $transport = new \Gelf\Transport\TcpTransport("<Docker container IP>", 12202);
    $publisher = new \Gelf\Publisher();

    $publisher->addTransport($transport);
    $monolog->pushHandler(new \Monolog\Handler\GelfHandler($publisher));
});
```

Xong, bây giờ bạn thực hiện lệnh `php artisan serve` và truy cập vào `localhost:8000` để sinh log rồi sang giao diện GrayLog và refresh lại trang rồi xem kết quả :D

<iframe width="560" height="315" src="https://www.youtube.com/embed/j3gcKOkPUyI" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# Lời kết

Đến đây là kết thúc bài viết của mình. Hy vọng qua bài viết này sẽ giúp mọi người có cái nhìn mới về cách quản lý log của ứng dụng. Bài viết trên đây chỉ là giới thiệu sơ qua về GrayLog và cách cài đặt cũng như cách sử dụng căn bản. Mọi người có thể tham khảo thêm các chi tiết nâng cao từ documentation của GrayLog tại [http://docs.graylog.org/en/2.3/](http://docs.graylog.org/en/2.3/).

Hẹn gặp lại mọi người ở bài viết sau, cũng là quản lý log ứng dụng giống thằng GrayLog này nhưng với bộ ba ứng dụng của Elastic là **ELK Stack** (**E**lasticSearch, **L**ogStash, **K**ibana) nhé :D!

Chào thân ái và quyết thắng (lol)!
