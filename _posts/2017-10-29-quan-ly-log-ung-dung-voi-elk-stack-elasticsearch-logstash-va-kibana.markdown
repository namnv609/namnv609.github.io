---
date: 2017-10-29 16:48:00 +0700
title: Quản lý log ứng dụng với ELK Stack (Elasticsearch, Logstash và Kibana)
---

Như phần trước mình đã giới thiệu về [**GrayLog 2 - quản lý log của ứng dụng**](/posts/quan-ly-log-ung-dung-voi-graylog-2.html). Hôm nay mình tiếp tục giới thiệu một bộ quản lý log ứng dụng đến từ [Elastic](https://elastic.co) là ELK Stack (**E**lasticsearch, **L**ogstash và **K**ibana) để các bạn có thêm lựa chọn khi muốn triển khai một ứng dụng quản lý log nhé.<!--more--> Bài viết giới thiệu về bộ ba ELK này đã có, bạn có thể đọc thêm [**tại đây**](https://viblo.asia/p/quan-ly-log-voi-logstash-elasticsearch-kibana-BAQ3vVj1vbOr) nên mình không giới thiệu thêm nữa mà đi luôn vào việc cài đặt và triển khai căn bản như thế nào nhé. Chúng ta lại thực hiện giống như với thằng GrayLog 2 là tạo một server bằng Docker nhé. Mặc định bạn đã cài đặt và sử dụng được Docker ở mức căn bản.

Đầu tiên, khởi tạo một container, chúng ta dùng Ubuntu 14.04 nhóe:

```
sudo docker run --name elk_stack --net=host -it ubuntu:14.04 /bin/bash
```

> Note: Việc dùng tham số `--net=host` là để sử dụng các port mà chúng ta mở trong container trực tiếp luôn trên host mà không cần phải đi qua IP của container

Sau khi đã khởi tạo và truy cập vào container, chúng ta thực hiện cài đặt một số packages cần thiết cho quá trình cài đặt ELK stack nhé. Đầu tiên là update lại APT cache
```
apt-get update
```

Sau đó, cài một số packages cần thiết

```
apt-get install -y wget curl vim software-properties-common python-software-properties unzip apt-transport-https
```

Sau khi đã cài các packages cần thiết cho quá trình cài đặt, chúng ta bắt đầu cài đặt các thành phần của ELK server.

# Cài đặt ELK Stack
## Cài đặt Java 8

Elasticsearch và Logstash yêu cầu phải có Java, nên chúng ta sẽ cài đặt nó đầu tiên. Trước hết, thêm APT repository của Java 8:

```
add-apt-repository -y ppa:webupd8team/java
```

Update lại APT cache

```
apt-get update
```

Và cài Java

```
apt-get -y install oracle-java8-installer
```

Sau khi cài Java xong, chúng ta chuyển sang cài Elasticsearch

## Cài đặt Elasticsearch

Để có thể cài đặt Elasticsearch thông qua APT, chúng ta cần thêm Elasticsearch vào APT source list. Đầu tiên, chúng ta import GPG key của Elasticsearch

```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```

Thêm Elasticsearch source list

```
echo "deb http://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
```

Cập nhật lại APT cache

```
apt-get update
```

Cài đặt Elasticsearch

```
apt-get install -y elasticsearch
```

Sau khi cài đặt xong, bạn thử xem Elasticsearch đã hoạt động chưa bằng lệnh:

```
curl localhost:9200
```

Bạn sẽ nhận được kết quả dạng:

```json
{
  "name" : "Agent X",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.3.2",
    "build_hash" : "b9e4a6acad4008027e4038f6abed7f7dba346f94",
    "build_timestamp" : "2016-04-21T16:03:47Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.0"
  },
  "tagline" : "You Know, for Search"
}
```
Nếu nó chưa start, bạn chạy lệnh:

```
service elasticsearch start
```

Sau khi cài xong Elasticsearch, chúng ta sang phần cài Kibana nhé.

## Cài đặt Kibana

Thêm Kibana source list cho APT

```
echo "deb http://packages.elastic.co/kibana/4.4/debian stable main" | sudo tee -a /etc/apt/sources.list.d/kibana-4.4.x.list
```

Update lại APT cache

```
apt-get update
```

Sau đó, cài đặt Kibana thông qua APT

```
 apt-get install -y kibana
```

Start Kibana bằng lệnh:

```
service kibana start
```

Và kiểm tra bằng cách truy cập vào địa chỉ: [http://localhost:5601](http://localhost:5601). OK, chúng ta đã cài xong Kibana, giờ chúng ta chuyển sang cài đặt Logstash

## Cài đặt Logstash

Thêm Logstash source list cho APT

```
echo 'deb http://packages.elastic.co/logstash/2.2/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash-2.2.x.list
```

Cập nhật lại APT cache

```
apt-get update
```

Cài đặt Logstash

```
apt-get install -y logstash
```

Sau khi cài đặt xong, khởi động Logstash bằng lệnh:

```
service logstash start
```

Sau khi cài đặt xong, chúng ta thực hiện cấu hình cho Logstash nhóe.

## Cấu hình Logstash

Đầu tiên, chúng ta tạo file `02-beats-input.conf` cho Filebeat

```
vi /etc/logstash/conf.d/02-beats-input.conf
```

Và thêm nội dung sau:

```
input {
  beats {
    port => 5044
  }
}
```

Tiếp theo, chúng ta sẽ tạo filter cho Syslog để test:

```
vi /etc/logstash/conf.d/10-syslog-filter.conf
```

Thêm nội dung bên dưới vào file `10-syslog-filter.conf`

```
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}
```

Tiếp theo, chúng ta sẽ cấu hình output ra Elastichsearch

```
vi /etc/logstash/conf.d/30-elasticsearch-output.conf
```

Thêm nội dung dưới vào file `30-elasticsearch-output.conf`

```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    sniffing => true
    manage_template => false
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
  }
}
```

Sau khi đã hoàn thành, chúng ta thử kiểm tra xem các cài đặt có đúng hay không bằng lệnh

```
service logstash configtest
```

Khởi động lại Logstash để nó cập nhật các cài đặt mới

```
service logstash restart
```

Nếu mọi thứ đều đúng, bạn sẽ nhận được message sau: `Configuration OK`. Chúng ta sang bước tiếp, tải Kibana Dashboards sample

```
cd ~
curl -L -O https://download.elastic.co/beats/dashboards/beats-dashboards-1.1.0.zip
```

Sau khi tải xong, chúng ta unzip nó ra

```
unzip beat-dashboards-*.zip
```

Sau đó, load các sample vào Elasticsearch

```
cd beats-dashboards-*
./load.sh
```

Sau khi chạy xong, bạn truy cập [http://localhost:5601](http://localhost:5601) sẽ thấy có các index patterns sau:

* filebeat-*
* packagebeat-*
* topbeat-*
* winlogbeat-*

Tiếp theo, chúng ta sẽ import Filebeat index template cho Elasticsearch

```
cd ~
curl -O https://gist.githubusercontent.com/thisismitch/3429023e8438cc25b86c/raw/d8c479e2a1adcea8b1fe86570e42abab0f10f364/filebeat-index-template.json
curl -XPUT 'http://localhost:9200/_template/filebeat?pretty' -d@filebeat-index-template.json
```

Nếu thành công, bạn sẽ nhận được thông báo sau:

```
{
  "acknowledged" : true
}
```

Vậy là đã xong phần server chứa log. Bây giờ server ELK stack của chúng ta có thể nhận log từ các server chứa ứng dụng. Bây giờ chúng ta quay về host để cài Filebeat và thử gửi log xem sao nhé

## Cài đặt Filebeat

> Phần này chúng ta thực hiện trên host chứ không phải trong Docker container nhé

Đầu tiên, chúng ta APT source list cho Filebeat

```
echo "deb https://packages.elastic.co/beats/apt stable main" |  sudo tee -a /etc/apt/sources.list.d/beats.list
```

Import GPG key của Elasticsearch

```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```

Cập nhật lại APT cache và cài đặt Filebeat

```
sudo apt-get update
sudo apt-get install filebeat
```

Sau khi cài đặt xong, chúng ta cấu hình cho Filebeat.

```
sudo vi /etc/filebeat/filebeat.yml
```

Phần log paths, chúng ta không cần gửi tất cả các file `*.log` mà chỉ thử hai file là `auth.log` và `syslog` nên chúng ta sẽ xóa key `/var/log/*.log` rồi thêm hai key sau:

```yaml
# ...
            paths:
                - /var/log/auth.log
                - /var/log/syslog
# ...
```

Tiếp đến, chúng ta đổi key `document_type` từ `log` sang `syslog`

```yaml
# ...
            document_type: syslog
# ...
```

Phần `hosts` của `logstash` chúng ta không cần sửa vì mặc định nó là `localhost:5044` rồi (và do chúng ta sử dụng tùy chọn `--net=host` khi chạy Docker nên các port được mở ra sẽ bind trực tiếp vào localhost của host), còn trong thực tế, bạn hãy sửa nó thành IP (hoặc domain) của server chứa ELK stack nhé. Chúng ta thêm key sau:

```yaml
# ...
        logstash:
            hosts: ["localhost:5044"]
            bulk_max_size: 1024
# ...
```

Sau khi xong, chúng ta khởi động lại Filebeat

```
sudo service filebeat restart
```

Xong, giờ chúng ta thử xem cài đặt của Filebeat đã chạy chưa bằng cách quay lại Kibana tại [http://localhost:5601](http://localhost:5601) và chọn index pattern `filebeat-*` để xem nhé.

# Lời kết

Vậy là mình đã giới thiệu xong phần cài đặt và chạy thử bộ ELK stack. Hy vọng bài viết sẽ giúp bạn có thêm sự lựa chọn trong việc quản lý log của ứng dụng. Hẹn gặp lại mọi người ở bài viết khác (seeyou)
