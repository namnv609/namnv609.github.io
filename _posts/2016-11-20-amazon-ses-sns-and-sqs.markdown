---
date: 2016-11-20 07:38:00 +0700
title: Amazon SES, SNS and SQS
---

Trong dự án hiện tại mình đang tham gia, mình có cơ hội được sử dụng, tiếp cận và tìm hiểu các dịch vụ của Amazon Web Services (AWS). Và có 03 dịch vụ mình tập trung vào tìm hiểu nhiều nhất là Amazon SES (**S**imple **E**mail **S**ervice), Amazon SNS (**S**imple **N**otification **S**ervice) và Amazon SQS (**S**imple **Q**ueue **S**ervice).<!--more--> Lý do vì sao? Vì dự án hiện tại cần sử dụng rất nhiều đến việc gửi email. Không chỉ đơn giản là gửi email kiểu thông báo xác nhận đăng ký thành viên hay dạng email chào mừng mỗi khi có một người dùng nào đó đăng ký. Mà nó dạng email marketing, cần phải gửi số lượng lớn, cần phải tracking rằng email đó có đến được với người dùng hay không, người dùng đó thao tác như thế nào với email của mình đã gửi đến cho họ, ...!

Với những yêu cầu ở trên, tại sao dự án mình lại không sử dụng một dịch vụ email marketing nào đó như [MailChimp](https://mailchimp.com/), [CampaignMonitor](http://www.campaignmonitor.com/), [SendGrid](https://sendgrid.com/), ...? Lý do thì mình không biết vì sao đâu. Nhưng được yêu cầu thì mình cứ tìm hiểu và áp dụng thôi (yaoming). Nhưng cũng cho đấy là sự may mắn đối với mình, được có cơ hội tìm hiểu về Amazon Web Services, làm sao để tracking email với các dịch vụ sẵn có của AWS và cũng có cái nhìn cơ bản về cách xây dựng một email marketing service (có lẽ nào nên xây dựng một hệ thống để cạnh tranh với các dịch vụ email marketing sẵn có nhỉ =)))!

OK, đã có phần mở bài cơ bản, giờ chúng ta sẽ đi tìm hiểu qua về 03 dịch vụ mà mình đã giới thiệu ở trên nhé. Để biết từng dịch vụ sẽ được áp dụng và sử dụng như thế nào để có được một cái cơ bản về email marketing nhé (cảm giác đao to búa lớn ác (yaoming))!

### Amazon SES (Simple Email Service)

Amazon SES là gì? Nó là một dịch vụ cung cấp cho ta một server SMTP hỗ trợ việc gửi email với số lượng lớn (tuy nhiên, Amazon SES không hỗ trợ chúng ta gửi email trực tiếp trên Amazon mà cần phải sử dụng một công cụ gửi email nữa như các phần mềm gửi email, hay thông qua API của Amazon), hỗ trợ chúng ta thống kê về kết quả của email gửi đi thông qua 04 tham số như Deliveries, Bounces, Rejects và Complaints. Vậy, 04 tham số kia là gì? Và tại sao dự án của mình lại cần tới 03 dịch vụ để xử lý cho việc gửi email số lượng lớn? Chúng ta tìm hiểu qua chút nhé.

* **Deliveries**: Số lượng email được gửi đi trong ngày.
* **Bounces**: Số lượng email không tồn tại (không có thật) hoặc không nhận được email từ Amazon (dung lượng hòm thư đã đủ, server của người nhận email có vấn đề, ...).
* **Rejects**: Số lượng những email bị từ chối.
* **Complaints**: Số lượng email bị đánh dấu là Spam.

04 tỉ lệ trên có gì quan trọng không? Thưa các bạn là có, vì Amazon SES có những tiêu chí đánh giá tài khoản (để khóa, cảnh cáo, ...) theo các tỉ lệ đó. Ví dụ nếu tỉ lệ email không tồn tại (bounces) lớn hơn 5% và tỉ lệ bị đánh dấu là spam (Complaints) vượt quá 0.1% thì tài khoản của bạn sẽ bị khóa vì nó coi đó là dấu hiệu của spammer. Nên bạn cần phải cân nhắc cẩn thận. Và đó cũng là lý do vì sao mình lại cần tới 02 dịch vụ phụ trợ là SNS và SQS. Lý do vì sao mình sẽ giải thích sau. Giờ chúng ta đi tìm hiểu tiếp 02 dịch vụ còn lại nhé.

### Amazon SNS (Simple Notification Service)

Amazon SNS là một dịch vụ cho phép bạn gửi tin nhắn (SMS), thông báo (notification) số lượng lớn tới các thiết bị đầu cuối hay các client thông qua một topic topic. Các thiết bị đầu cuối (hay client) có thể là một web server (HTTP/S), email, Amazon SQS (mình sẽ giới thiệu sau) hay AWS Lambda. Vậy nó có chức năng gì khi mình sử dụng nó với Amazon SES? Nó sẽ là thành phần nhận thông báo về trạng thái của một email đã được gửi đi thông qua SES (như delivery, bounce, complaint hay reject).

### Amazon SQS (Simple Queue Service)

> Amazon Simple Queue Service (SQS) là một dịch vụ hàng đợi (queue) lưu trữ thông điệp (message) nhanh chóng, đáng tin cậy, có khả năng mở rộng và quản lý một cách đầy đủ. Amazon SQS giúp bạn có thể di chuyển dữ liệu giữa các thành phần phân tán của ứng dụng của bạn để thực hiện các nhiệm vụ khác nhau. Bạn có thể sử dụng SQS để truyền tải bất kỳ khối lượng dữ liệu, ở bất kỳ mức độ thông lượng nào mà không sợ bị mất đi các thông điệp hoặc yêu cầu của mỗi thành phần là luôn luôn có sẵn.
>
> Với SQS, bạn có thể giảm tải gánh nặng cho hệ thống và có thể dễ dàng mở rộng thông điệp lưu trữ, trong khi chỉ phải trả chi phí thấp cho những gì bạn sử dụng.

_Nội dung trong phần giới thiệu về SQS mình sử dụng lại trong bài [Tìm hiểu về Amazon SQS](https://viblo.asia/vuong.xuan.hung/posts/3OEqGjylM9bL) của bạn [Vương Xuân Hưng](https://viblo.asia/users/show/vuong.xuan.hung). Xin chân thành cảm ơn bạn._

Amazon SQS sẽ có chức năng gì trong bộ ba SES, SNS và SQS mà mình sẽ giới thiệu? Là nó sẽ được sử dụng để lưu trữ các notification thông báo về trạng thái mà thằng SNS trả về mỗi khi có một email của SES được gửi đi. Lý do vì sao? Vì theo các giao thức mà SNS hỗ trợ gồm có web server (là một endpoint mà chúng ta xây dựng trên chính server của mình để tiếp nhận notification) hay email, ... thì nó không được đảm bảo rằng server của chúng ta sẽ luôn online để nhận các notification đó. Hoặc số lượng email gửi đi rất lớn thì server của chúng ta cũng sẽ bị SNS gửi rất nhiều request. Nên sẽ dẫn đến có trường hợp không mong muốn là bị lack các notification. Vậy chúng ta mới sử dụng SQS để lưu trữ các notification đó. Server của chúng ta sẽ chủ động request lên SQS để nhận các notification theo định kỳ hoặc trong khoảng thời gian nhàn rỗi. Để tránh sự quá tải cho server và tránh được việc mất các notification trong một số trường hợp đặt biệt.

Mình đã giới thiệu xong 03 dịch vụ mà mình sẽ sử dụng, giờ chúng ta sẽ đi vào cách cài đặt cho 3 dịch vụ này có thể hiểu và làm việc với nhau nhé.

### Cài đặt

Đầu tiên, các bạn đăng ký dịch vụ Amazon SES, xác nhận domain mà chúng ta sẽ sử dụng theo các hướng dẫn cài đặt của SES. Rất tiếc là mình không có cơ hội để làm lại các bước xác nhận domain trên SES để chụp ảnh lại chi tiết. Các bạn có thể xem hướng dẫn [**tại đây**](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-domains.html)!

Tiếp theo, các bạn đăng ký sử dụng dịch vụ SNS. Sau đó tạo 1 topic với tên mà bạn mong muốn. Bạn nhập **Topic name** rồi nhấn **Create topic**!

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 15-55-56.png](/assets/images/posts/7cd8f3ef-c982-4bc0-bc7c-c0513c331ab4.png)

Tiếp tục, bạn đăng ký Amazon SQS. Sau đó tạo một queue để lưu trữ các notification do SNS gửi về.

![Screenshot from 2016-11-20 15:59:23.png](/assets/images/posts/d9abf60e-f81c-4d31-b6c7-77711a5a617a.png)

Các bạn có thể chọn kiểu queue là **Standard** hoặc **FIFO** (**F**irst **I**n **F**irst **O**ut). Hai kiểu queue này khác nhau như thế nào? Kiểu standard là khi bạn gọi lên SQS thông qua API, nó sẽ trả về queue message ngẫu nhiên có trọng số. Còn kiểu FIFO là nó trả về cho bạn theo thứ tự, cái nào tới trước nó sẽ trả ra trước. Sau khi nhập **Queue name**, các bạn bấm **Quick-Create Queue** nếu muốn sử dụng cài đặt mặc định. Còn không, các bạn bấm vào **Configure Queue** để cài đặt thêm theo ý muốn.

Sau khi đã tạo xong SES, SNS và SQS. Ở màn hình danh sách các queue, các bạn chọn queue mà mình vừa tạo rồi bấm **Queue Actions** > **Subscribe Queue to SNS Topic**

![Screenshot from 2016-11-20 16:09:31.png](/assets/images/posts/c18cdbfe-2b3e-48c8-a806-e471b41dac09.png)

Chọn topic mà bạn vừa tạo ở bên SNS trong phần **Choose a Topic** rồi bấm **Subscribe**. Vậy là bạn đã xong phần subscribe SQS để nhận notification từ SNS.

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 16-34-50.jpg](/assets/images/posts/b8cd702a-61f3-4bf0-9aa4-83209f77bc64.jpg)

Tiếp theo, chúng ta sẽ cài đặt cho SES gửi email status về SNS. Sau khi đã verify domain ở bên SES, các bạn verify một email sử dụng để gửi đi (trong phần From mà người dùng sẽ nhận được). Chọn menu **Email Addresses** > **Verify a New Email Address**

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 16-16-55.png](/assets/images/posts/b03ee45f-e8bd-4ff5-91f8-2b1e21b6926c.png)

Nhập địa chỉ email vào phần **Email Address** và bấm **Verify This Email Address**. Sau đó vào hòm mail mà bạn vừa nhập để xác nhận.

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 16-18-46.png](/assets/images/posts/0d180a6b-c490-4671-b5ca-c3b4e390bf92.png)

Sau khi xác nhận xong. Ở danh sách, các bạn vào chi tiết của địa chỉ email đó. Phần **Notification** các bạn chọn **Edit Configuration** để cài đặt SES với SNS.

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 16-25-49.jpg](/assets/images/posts/1d074ff2-61a0-4306-b84f-7967ec887166.jpg)

Ở phần **Bounces**, **Complaints** và **Deliveries** các bạn chọn SNS topic mà mình vừa tạo. Check vào **Include original headers** nếu bạn muốn nhận notification bao gồm cả email headers. Phần **Email Feedback Forwarding** các bạn có thể chọn **Disabled** nếu không muốn nhận notification về địa chỉ email mà bạn vừa xác nhận. Bấm **Save Config** để lưu cài đặt.

![screenshot-us-west-2.console.aws.amazon.com 2016-11-20 16-29-23.jpg](/assets/images/posts/62356baa-d1e0-4ee4-84ab-d3efcfc8e9d4.jpg)

Vậy là đã xong phần cài đặt cho 03 dịch vụ mà mình đã giới thiệu. Giờ chúng nó có thể hiểu và làm việc với nhau. Mình xin tạm dừng phần 1 tại đây. Sang phần 2, mình sẽ đi vào chi tiết làm sao để xử lý các email status notification do SES gửi cho SNS thông qua thằng trung gian là SQS nhé :D!
