---
date: 2016-11-26 09:11:00 +0700
title: Amazon SES, SNS and SQS (Part II)
---

Như trong [**Phần I**](/posts/amazon-ses-sns-and-sqs.html) mình đã giới thiệu và hướng dẫn cách cài đặt cho 03 dịch vụ có thể làm việc được với nhau cho mục đích tracking email status.<!--more--> Trong phần II này, mình sẽ đi vào chi tiết làm sao để xử lý các thông tin mà 03 dịch vụ này cung cấp. Do dự án hiện tại mình đang tham gia sử dụng ngôn ngữ **Ruby** (và sử dụng Rails framework) nên mình sẽ dùng code **Ruby** để thực hiện việc xử lý nhé. Mình sẽ chia thành hai kiểu là **xử lý thụ động** và **xử lý chủ động** để mọi người có thể lựa chọn cho phù hợp với mục đích sử dụng của mình.

## Xử lý thụ động

Xử lý thụ động là gì? Là chúng ta sẽ thụ động lắng nghe những thông tin mà Amazon SNS trả về cho server của mình mỗi khi nó nhận được một email status do chúng ta gửi đi thông qua dịch vụ Amazon SES.

Ưu điểm của việc xử lý này là chúng ta sẽ xử lý được luôn mỗi khi có một email nào đó gửi đi và nhận được trạng thái của email đó và chúng ta cũng không cần phải sử dụng đến dịch vụ Amazon SQS (nếu thực sự bạn thấy không cần thiết). Nhược điểm của nó là bạn cần phải đầu tư về phần cứng và hạ tầng để xử lý cũng như tránh việc bị lack thông tin do Amazon SNS gửi về trong trường hợp bạn gửi email số lượng lớn. Để test việc xử lý này trên local cho tiện việc debug và kiểm tra, mình sẽ sử dụng phần mềm [**Ngrok**](https://ngrok.com/) để có thể public localhost ra internet.

Lý do tại sao chúng ta lại cần phải public localhost ra internet? Vì việc xử lý thụ động này là chúng ta sẽ tạo một endpoint với danh nghĩa là 1 subscription của SNS. Mỗi khi có 1 trạng thái của email nào đó từ SES gửi tới SNS, SNS sẽ thực hiện việc gửi notification cho chúng ta xử lý thông qua đường dẫn mà chúng ta đã đăng ký cho topic trên SNS. Cách tạo 1 topic trên SNS, các bạn có thể xem lại hướng dẫn ở phần I. Giờ chúng ta đi vào phần code trước khi đăng ký 1 subscription trong SNS topic nhé :D. Mình sẽ cố gắng giải thích chi tiết nhất có thể :D. Let's go!!!!!!

Đầu tiên, chúng ta sẽ tạo 1 url trong `config/routes.rb` nhé. Mình lấy tạm cái tên là `sns_notifications`:

```ruby
Rails.application.routes.draw do
  # ...
  post "/sns_notifications", to: "sns_notifications#index", as: :sns_notifications
  # ...
end
```

Tiếp đến, chúng ta tách phần xử lý notification từ SNS thành 1 service cho dễ quản lý nhé :D

```ruby
# app/services/amazon_sns_service.rb
class AmazonSnsService
  # Sử dụng net/http để thực hiện tạo request confirm subscription cho Amazon SNS topic
  require "net/http"

  # Khởi tạo 1 số constant để định nghĩa cho SES status
  # Do dự án mình không quan tâm đến reject status nên mình không định nghĩa
  # nếu bạn cần, bạn có thể triển khai thêm nhé ^^
  BOUNCE_STATUS = "Bounce"
  COMPAINT_STATUS = "Complaint"
  DELIVERY_STATUS = "Delivery"

  # Khởi tạo service
  # Nhận tham số là 1 request body (được gửi từ SNS)
  def initialize request
    # Lấy SNS topic ARN. Mình làm vậy là để tránh việc người khác
    # sử dụng endpoint của mình bằng việc kiểm tra xem topic ARN đó
    # có phải là của mình hay không. Nếu đúng thì mới xử lý tiếp
    # còn không thì bơ đi :D
    @sns_topic_arn = request.env["HTTP_X_AMZ_SNS_TOPIC_ARN"]
  end

  # Kiểm tra xem topic ARN đó có phải là của mình hay không
  def check_valid_request?
    # Khởi tạo danh sách các topic ARN được phép xử lý
    allowed_arn_topics = %w(arn:aws:sns:sns-region-name:123456789:topic-name)

    allowed_arn_topics.include? @sns_topic_arn
  end

  # Xác nhận với Amazon SNS rằng URL endpoint của chúng ta là chính chủ
  # và nó đang hoạt động chứ không phải là điền bừa URL endpoint =)
  # Params:
  # +subscription_url+: Amazon SNS confirmation subscription URL
  def confirmation_subscription subscription_url
    # Việc confirm rất đơn giản. Chúng ta chỉ cần tạo 1 GET request
    # đến URL mà Amazon trả về cho chúng ta. Nên chúng ta sẽ sử dụng
    # Net::HTTP để thực hiện tạo request nhé
    response = Net::HTTP.get_response(URI.parse(subscription_url))

    # Ghi log lại cho tiện việc kiểm tra :D
    Rails.logger.info(
      "Amazon SNS Service: Confirmation status: #{response.code}"
    )
  rescue StandardError => http_get_response_exception
    # Đề phòng trường hợp request lỗi, cũng cần phải ghi log lại
    # để tiện cho việc kiểm tra và xử lý
    Rails.logger.error(
      "Amazon SNS Service Subscription confirmation error: #{http_get_response_exception}"
    )
  end

  # Tiếp đến, chúng ta sẽ xử lý email status mà SNS gửi cho chúng ta nhé
  # Phần xử lý này mình làm theo tính chất của dự án
  # các bạn hãy thay đổi việc xử lý cho phù hợp với mục đích của mình nhé
  # Params:
  # +notification_message+: Amazon SNS notification message
  def update_email_status notification_message
    # Tính chất của method này (đối với mình) chỉ đơn giản là update
    # cột email_status của User thành true hoặc false cho các trạng thái khác nhau
    # nên method này của mình nó rất đơn giản :D và mục đích sử dụng các email status
    # cũng khá khác so với gì mình đã giới thiệu ở phần I nhé
    case notification_message[:notificationType]
    when BOUNCE_STATUS
      update_user_email_status notification_message[:mail], false
    when COMPLAINT_STATUS, DELIVERY_STATUS
      update_user_email_status notification_message[:mail], true
    end
  end

  private
  # Tạo private method để update user email status nhé :D
  # Params:
  # +mail_object+: Amazon SNS mail object
  # +email_status+: User email status
  def update_user_email_status mail_object, email_status
    # Lấy địa chỉ email từ mail_object
    user_emails = mail_object.try(:[], "destination")
    # Không xử lý gì nếu mail_object và user_emails không hợp lệ
    return unless mail_object && user_emails

    # Update email_status theo status mà chúng ta nhận được từ method trước đó
    User.where(email: user_emails).update_all email_status: email_status
  end
end
```

Xong phần service xử lý, tiếp đến, chúng ta tạo controller tiếp nhận request từ SNS và gọi service để xử lý.

```ruby
# app/controllers/sns_notifications_controller.rb
class SnsNotificationsController < ApplicationController
  # Khởi tạo 1 số constants cho nó có nghĩa
  AMAZON_SUBSCRIPTION_TYPE = "SubscriptionConfirmation"
  AMAZON_UNSUBSCRIPTION_TYPE = "UnsubscribeConfirmation"
  AMAZON_NOTIFICATION_TYPE = "Notification"

  # Bỏ qua việc kiểm tra CSRF để có thể nhận dữ liệu được post từ SNS
  skip_before_action :verify_authenticity_token, only: [:index]

  # Thực hiện việc xử lý
  def index
    # Nhận request body từ SNS
    request_body = get_request_body request.raw_post
    # Khởi tạo service
    amazon_sns_svc = AmazonSnsService.new request_body

    # Kiểm tra xem request_body và ARN topic có hợp lệ hay không
    # mới tiếp tục xử lý tiếp. Còn không thì lướt
    if request_body.any? && amazon_sns_svc.valid_request?
      # Kiểm tra xem kiểu notification mà SNS trả về cho chúng ta là gì
      # để chúng ta sẽ xử lý bằng các method tương ứng trong service
      # mà chúng ta vừa viết :D
      case request_body[:Type]
      when AMAZON_SUBSCRIPTION_TYPE, AMAZON_UNSUBSCRIPTION_TYPE
        # Nếu request là confirm subscription hay unsubscription
        # thì chúng ta sẽ gọi method confirmation_subscription trong service
        amazon_sns_svc.confirmation_subscription(
          request_body[:SubscribeURL]
        )
      when AMAZON_NOTIFICATION_TYPE
        # Nếu request là notification, chúng ta sẽ lấy notification message để xử lý
        notification_message = get_request_body request_body[:Message]

        # Kiểm tra xem có message trong request_body hay không
        # nếu có, chúng ta sẽ update email status theo những gì
        # mà SNS trả về cho chúng ta
        if notification_message.any?
          amazon_sns_svc.update_email_status notification_message
        end
      end
    end

    # Render ra 1 trang trắng sau khi xử lý :D
    render nothing: true
  end

  private
  # Tạo private method để parse request body
  # Params:
  # +request_body+:: Raw request body
  def get_request_body request_body
    # Object mặc định, phòng trường hợp request body không phải là
    # một chuỗi JSON hợp lệ
    request_body_object = {}

    # Bắt đầu parse dữ liệu
    begin
      request_body_object = JSON.parse request_body
    rescue JSON::ParserError => json_parse_error
      # request_body không phải là một chuỗi JSON hợp lệ, ghi log lại
      logger.error(
        "SNS Notifications Controller parse request: #{json_parse_error}"
      )
    end

    # Trả request_body dạng symbolize_keys
    request_body_object.symbolize_keys
  end
end
```

Vậy là xong phần logic xử lý thụ động. Khá đơn giản phải không :D! Giờ chúng ta đi vào việc thử xem nó làm việc có ổn không nhé :D.

Đầu tiên, chúng ta chạy dự án bằng lệnh ``rails server`` để mặc định trên cổng 3000. Tải Ngrok [**tại đây**](https://ngrok.com/download). Sau đó xả nén rồi phân quyền thực thi cho Ngrok bằng lệnh: ``chmod +x path/to/ngrok`` và chạy Ngrok để forward từ domain của nó về localhost ở cổng 3000 của mình bằng lệnh: ``./path/to/ngrok http 3000`` và chúng ta sẽ có màn hình sau:

```
ngrok by @inconshreveable (Ctrl+C to quit)

Session Status                online
Version                       2.1.18
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://<Random ID>.ngrok.io -> localhost:3000
Forwarding                    https://<Random ID>.ngrok.io -> localhost:3000

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

Vào Amazon SNS topic mà bạn đã đăng ký (xem lại **Phần I**), tạo subscription với dạng HTTP (hoặc HTTPS nếu muốn):

![screenshot-us-west-2.console.aws.amazon.com 2016-11-26 17-19-23.jpg](/assets/images/posts/465c1ccb-d091-4388-9937-03af7ca961fb.jpg)

Các bạn điền thông tin như trong hình (phần **Endpoint** các bạn lấy từ phần URL mà Ngrok trả cho chúng ta nhé) rồi bấm **Create subscription**. Nếu như endpoint của chúng ta hoạt động đúng, sau khi bạn F5 lại trang list subscription trên Amazon, bạn sẽ nhận được như bên dưới:

![screenshot-us-west-2.console.aws.amazon.com 2016-11-26 17-32-48.jpg](/assets/images/posts/6d4a035c-2787-409b-a786-8f4c6b41cae4.jpg)

```
I, [2016-11-26T17:31:19.397680 #7563]  INFO -- : Started POST "/sns_notifications" for 54.240.230.245 at 2016-11-26 17:31:19 +0700
I, [2016-11-26T17:31:19.598568 #7563]  INFO -- : Processing by SnsNotificationsController#index as HTML
I, [2016-11-26T17:31:21.040451 #7563]  INFO -- : Amazon SNS Service: Confirmation status: 200
I, [2016-11-26T17:31:21.065633 #7563]  INFO -- :   Rendered text template (0.0ms)
I, [2016-11-26T17:31:21.065954 #7563]  INFO -- : Completed 200 OK in 1467ms (Views: 25.1ms | ActiveRecord: 0.0ms)
```

```
ngrok by @inconshreveable (Ctrl+C to quit)

Session Status                online
Version                       2.1.18
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://<Random ID>.ngrok.io -> localhost:3000
Forwarding                    https://<Random ID>.ngrok.io -> localhost:3000

Connections                   ttl     opn     rt1     rt5     p50     p90
                              1       0       0.00    0.00    26.77   26.77
HTTP Requests
-------------

POST /sns_notifications        200 OK
```

Vậy là xong phần confirm. Bây giờ chúng ta thử gửi một email để test trạng thái nhé :D! Sang Amazon SES, chọn email mà bạn đã verify cùng với việc cài đặt notification là SNS topic đã tạo sẵn, click vào **Send a Test Email**.

![screenshot-us-west-2.console.aws.amazon.com 2016-11-26 17-52-46.jpg](/assets/images/posts/94f514e0-c815-404f-b10d-064617a6c188.jpg)

Nhập các thông tin cần thiết. Riêng phần **To**, bạn có thể tham khảo thêm các địa chỉ email test khác mà Amazon SES cung cấp cho chúng ta [**tại đây**](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/mailbox-simulator.html)! Mỗi email lại có 1 status khác nhau. Và việc sử dụng email test này sẽ không tác động đến send quota mà Amazon SES cung cấp cho chúng ta (trong sandbox mode là 200 emails/24hours). Sau khi bấm **Send Test Email**, nếu code chúng ta hoạt động đúng, bạn sẽ thấy log sau:

```
I, [2016-11-26T17:44:02.208662 #7563]  INFO -- : Started POST "/sns_notifications" for 54.240.230.241 at 2016-11-26 17:44:02 +0700
I, [2016-11-26T17:44:02.244445 #7563]  INFO -- : Processing by SnsNotificationsController#index as HTML
D, [2016-11-26T17:44:02.273953 #7563] DEBUG -- :   SQL (1.9ms)  UPDATE `users` SET `users`.`email_status` = 0 WHERE `users`.`email` = 'bounce@simulator.amazonses.com'
I, [2016-11-26T17:44:02.275107 #7563]  INFO -- :   Rendered text template (0.0ms)
I, [2016-11-26T17:44:02.275416 #7563]  INFO -- : Completed 200 OK in 31ms (Views: 0.6ms | ActiveRecord: 11.3ms)
D, [2016-11-26T17:59:38.182155 #7563] DEBUG -- :
D, [2016-11-26T17:59:38.182315 #7563] DEBUG -- :
I, [2016-11-26T17:59:38.182503 #7563]  INFO -- : Started POST "/sns_notifications" for 54.240.230.241 at 2016-11-26 17:59:38 +0700
I, [2016-11-26T17:59:38.204126 #7563]  INFO -- : Processing by SnsNotificationsController#index as HTML
D, [2016-11-26T17:59:38.220882 #7563] DEBUG -- :   SQL (0.6ms)  UPDATE `users` SET `users`.`email_status` = 1 WHERE `users`.`email` = 'success@simulator.amazonses.com'
I, [2016-11-26T17:59:38.221206 #7563]  INFO -- :   Rendered text template (0.0ms)
I, [2016-11-26T17:59:38.221395 #7563]  INFO -- : Completed 200 OK in 17ms (Views: 0.3ms | ActiveRecord: 0.6ms)
```

Vậy là xong phần **xử lý thụ động**. Mọi việc đã thành công tốt đẹp. Bạn đã hiểu được phần xử lý email status đơn giản (yeah)! Bài viết đến đây cũng đã dài, sang phần sau, mình sẽ tiếp tục giới thiệu về phần **xử lý chủ động** để mọi người có thêm lựa chọn nhé.

Hy vọng mọi người sẽ hiểu những gì mà mình đã chia sẻ (văn mình hơi lủng củng xíu (yaoming))! Hẹn gặp lại mọi người ở phần III (chao)!
