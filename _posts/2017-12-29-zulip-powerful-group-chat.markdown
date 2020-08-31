---
date: 2017-12-29 08:14:00 +0700
title: Zulip - Powerful group chat
---

Hôm nay, mình sẽ giới thiệu tới các bạn các bước tự cài đặt một group chat với nhiều tính năng mạnh mẽ như Slack. Đó là [**Zulip**](https://zulipchat.com/) - một group chat open source. Nếu bạn muốn có một group chat cho riêng team, công ty hay đơn giản là thích vọc vạch như mình. Thì bạn có thể thử. Chúng ta cùng đi tìm hiểu cách cài đặt và sử dụng cơ bản thằng Zulip này nhé.<!--more-->

# Cài đặt Zulip

Do đây là vọc vạch, nên chúng ta sử dụng Docker để cài đặt nhé. Mặc định máy mọi người đã cài đặt Docker và đã có thể sử dụng Docker căn bản. Cách cài đặt và sử dụng Docker như thế nào chúng ta không bàn tới ở đây nhé. Zulip hỗ trợ Ubuntu 16.04 Xenial và 14.04 Trusty (khuyến khích là 64-bit), nên chúng ta có thể pull một trong hai image đó để thử. OK, mình dùng image Ubuntu 14.04 nhé. Đầu tiên, pull image về (nếu bạn chưa có image này)

```
sudo docker pull ubuntu:14.04
```

Tiếp theo, tạo container từ image đã có

```
sudo docker run --name zulip_server -it ubuntu:14.04 /bin/bash
```

Sau khi khởi tạo và ở trong container. Đầu tiên là chúng ta cài đặt các packages cơ bản và cần thiết (do image này là bản minial) để có thể cài đặt Zulip nhé. Trước hết, update lại APT cache

```
apt-get update
```

Tiếp theo, cài đặt các packages cơ bản cần thiết

```
apt-get install -y curl wget vim software-properties-common python-software-properties
```

Sau khi cài đặt xong, chúng ta sẽ tạo chứng chỉ SSL cho Zulip. Vì dùng Docker và không có domain nên chúng ta không sử dụng được Let's Encrypt. Vậy chúng ta sẽ tạo một self-signed certificate nhé. Để tạo, chúng ta cần cài đặt `openssl` package

```
apt-get install -y openssl
```

Sau đó, bạn thực hiện các bước sau để tạo SSL

```
openssl genrsa -des3 -passout pass:x -out server.pass.key 4096
openssl rsa -passin pass:x -in server.pass.key -out zulip.key
rm server.pass.key
openssl req -new -key zulip.key -out server.csr
# Sau lệnh này, nó sẽ yêu cầu bạn nhập một số thông tin. Bạn có thể nhập tự do hoặc cứ Enter để bỏ qua nhé

openssl x509 -req -days 365 -in server.csr -signkey zulip.key -out zulip.combined-chain.crt
rm server.csr
mv zulip.key /etc/ssl/private/zulip.key
mv zulip.combined-chain.crt /etc/ssl/certs/zulip.combined-chain.crt
```

Vậy là đã xong khâu chuẩn bị. Bây giờ chúng ta đi cài đặt thằng Zulip nhé. Đầu tiên, chúng ta tạo thư mục tạm thời để chứa source code (hoặc không, bạn có thể tải về ở chỗ nào cũng được):

```
cd $(mktemp -d)
```

Tải và xả nén source

```
wget https://www.zulip.org/dist/releases/zulip-server-latest.tar.gz
tar -xf zulip-server-latest.tar.gz
```

Trước khi thực hiện cài đặt, chúng ta cần làm hai việc là cài đặt locale UTF-8 cho Python và trỏ lại hostname trong `/etc/hosts` để chạy RabbitMQ. Đầu tiên là cài đặt locale:

```
locale-gen "en_US.UTF-8"
dpkg-reconfigure locales
```

Sau đó, bạn thực hiện chạy lệnh sau để trỏ lại hostname cho RabbitMQ:

```
echo -e "127.0.0.1 $(hostname -s)\n$(cat /etc/hosts)" > /etc/hosts
```

Thực hiện cài đặt Zulip server

```
./zulip-server-*/scripts/setup/install
```

Quá trình cài đặt diễn ra tầm khoảng 15' (tùy vào tốc độ mạng). Bạn có thể nhâm nhi coffee trong lúc chờ đợi :D!

Sau khi cài đặt xong, trước khi tiến hành cài đặt Zulip, chúng ta cần phải có một SMTP mail, bạn có thể sử dụng Gmail, MailTrap, SendPulse, MailGun, ... ở đây mình dùng MailTrap. Giờ chúng ta bắt đầu tiến hành cấu hình cho Zulip bằng cách sửa các giá trị dưới đây ở file `/etc/zulip/settings.py`

```python
EXTERNAL_HOST = '<Docker IP>'
ZULIP_ADMINISTRATOR = '<Your email>'
EMAIL_HOST = 'smtp.mailtrap.io'
EMAIL_HOST_USER = '<MailTrap SMTP username>'
EMAIL_PORT = 2525
EMAIL_USE_TLS = False
```

Sau đó, sửa email password tại file `/etc/zulip/zulip-secrets.conf`

```
[secrets]
# Secrets config
email_password = <MailTrap SMTP password>
```

Sau khi cấu hình cho Zulip xong, chúng ta chuyển sang `zulip` user để thực hiện một số bước khởi tạo nhé

```
su - zulip
```

Đầu tiên, thử xem cài đặt cho SMTP của chúng ta đã thành công hay chưa bằng lệnh:

```
/home/zulip/deployments/current/manage.py send_test_email <Test email address>
```

Sau đó, bạn có thể kiểm tra lại Inbox ở MailTrap để xác nhận. Tiếp, chúng ta sẽ khởi tạo Zulip database:

```
/home/zulip/deployments/current/scripts/setup/initialize-database
```

Sau khi khởi tạo xong, khởi động lại Zulip service để nó áp dụng các cài đặt mà chúng ta đã thay đổi:

```
/home/zulip/deployments/current/scripts/restart-server
```

Tiếp đến, tạo Zulip organization:

```
/home/zulip/deployments/current/manage.py generate_realm_creation_link
```

Chúng ta sẽ nhận được message chứa link để tạo với dạng sau

```
Please visit the following secure single-use link to register your
new Zulip organization:

    https://<Docker IP>/create_realm/<Random hash>
```

Bạn thực hiện truy cập vào link mà Zulip cung cấp để thực hiện công việc khởi tạo.

Vậy là đã xong. Bây giờ bạn có thể khám phá em Zulip rồi đấy :D! Dưới đây là một số hình ảnh (bao gồm cả Web và Desktop application - do mình không có mobile device để thử nên không có ảnh) của em Zulip:

![Zulip Web](/assets/images/posts/80d16473-4c00-4704-84cf-f861bd1ae621.png)

![Zulip Desktop application](/assets/images/posts/097937e6-4770-4b0d-8cd0-8166220617a0.png)

![Zulip Login UI](/assets/images/posts/109b04d3-c3dc-48ae-a8c1-72783463f81e.png)

# Lời kết

Đến đây là bài viết của mình đã kết thúc. Hy vọng bài viết này sẽ giúp ích cho mọi người (trong tương lai). Bài viết có vẻ ngắn nhưng trong quá  trình cài đặt mình đã gặp rất nhiều lỗi và phải Google để giải quyết những vấn đề gặp phải mới có thể cài đặt xong em Zulip này. Hy vọng mọi người có thể cài đặt suôn sẻ theo bài hướng dẫn của mình. Hẹn gặp lại mọi người ở bài viết tiếp theo :D!

Link documentation: [https://zulip.readthedocs.io/](https://zulip.readthedocs.io/)
