---
date: 2016-12-26 12:45:00 +0700
title: Amazon SES, SNS and SQS (Part III)
---

Trong [**phần II**](/posts/amazon-ses-sns-and-sqs-part-ii.html) mình đã giới thiệu phần **xử lý thụ động** để thực hiện việc xử lý email status do Amazon SNS cung cấp cho chúng ta mỗi khi một email được gửi đi thông qua dịch vụ Amazon SES.<!--more--> Như mình đã nói, việc xử lý thụ động có ưu điểm là thực hiện update email status gần như tức thời mỗi khi email được gửi đi và phát sinh trạng thái (delivery, bounce, reject hay complaint). Nhưng có một nhược điểm là chúng ta cần phải có cấu hình server đủ mạnh để xử lý các request được gửi đến từ Amazon SNS để tránh việc mất thông tin do Amazon SNS chỉ (thử) gửi cho chúng ta 4 lần. Nếu 4 lần đó đều không được server chúng ta xử lý thì message đó sẽ bị mất đi. Hôm nay, mình sẽ giới thiệu tiếp phần **xử lý chủ động** để mọi người có thêm sự lựa chọn để chọn phương pháp phù hợp cho mục đích của mình nhé.

## Xử lý chủ động

Xử lý chủ động là gì? Là chúng ta sẽ sử dụng thêm dịch vụ Amazon SQS để lưu trữ message notification  thay vì nhận message notification trực tiếp từ Amazon SNS. Ưu điểm của phần xử lý này là chúng ta không cần phải quan tâm quá nhiều đến cấu hình server. Vì chúng ta sẽ chủ động tạo request lên Amazon SQS (bằng schedule) sau mỗi khoảng thời gian nhất định để lấy message notification. Nhược điểm của việc xử lý này là email status không được update tức thì mỗi khi có một email được gửi đi. Việc test phần xử lý này chúng ta không cần thêm các phần mềm hỗ trợ bên ngoài như phần xử lý thụ động kia (như Ngrok). OK, chúng ta sẽ đi vào phần cài đặt trước nhé.

Chúng ta vào dịch vụ Amazon SQS, tạo 1 queue để nhận message notification từ Amazon SNS. Phần tạo queue, các bạn có thể đọc lại [**phần I**](/posts/amazon-ses-sns-and-sqs.html) nhé :D! OK, xong xuôi rồi chúng ta thực hiện việc code nhé. Mình sẽ tạo thêm 1 service để thực hiện phần code xử lý này.

```ruby
# app/services/amazon_sqs_service
# encoding: UTF-8

class AmazonSqsService
  # Định nghĩa các email status mà chúng ta sẽ xử lý
  EMAIL_STATUSES = %i(Delivery Complaint Bounce)
  # Định nghĩa kiểu log. Chúng ta sẽ sinh log ra file riêng
  # để tiện cho việc theo dõi
  ALLOWED_LOG_TYPES = %i(info warn error debug)

  # Khởi tạo service
  def initialize
    # Khai báo queue URL. Queue URL bạn có thấy thấy trong dịch vụ
    # Amazon SQS sau khi khởi tạo 1 queue.
    queue_url = "<Queue URL>"
    # Khởi tạo poller options.
    # Các bạn có thể đọc thêm chi tiết tại:
    # http://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/QueuePoller.html phần Constructor Details
    poller_options = {
      # Thời gian tối đa để thực hiện 1 polling message. Vượt qua thời gian này, nó sẽ bỏ qua
      # và thực hiện 1 polling khác.
      # Mặc định là 20 giây
      wait_time_seconds: 20,
      # Có thực hiện xóa message sau mỗi lần xử lý hay không.
      # Chúng ta nên để false để tránh việc xử lý trùng lặp. Việc xóa hay không xóa
      # message sau mỗi khi xử lý, chúng ta có thể thực hiện trong polling block.
      # Mặc định false
      skip_delete: false,
      # Thời gian mà bạn muốn xử lý message. Nếu quá số giây quy định, message của bạn
      # chưa được xử lý kịp nó sẽ quay lại queue để bạn có thể lấy lại ở lần sau.
      # Mặc định là nil
      visibility_timeout: 60,
      # Thời gian giữ long polling. Nếu quá thời gian, long polling sẽ tự ngắt.
      # Mặc định là nil. Mình nghĩ bạn nên tính toán cẩn thận để tránh việc
      # chồng chéo các phiên làm việc với nhau
      idle_timeout: 20
    }
    # Khai báo số lượng queue message mà chúng là muốn lấy trong 1 lần.
    # Bạn nên tính toán cẩn thận để tránh việc chồng chéo các phiên làm việc với nhau
    max_poll_messages = 20
    # Update thông tin chứng chỉ của Amazon AWS
    Aws.config.update amazon_credential_configs
    # Khởi tạo QueuePoller
    @poller = Aws::SQS::QueuePoller.new queue_url, poller_options
    # Dừng long polling khi đạt được số message đã chỉ định
    stop_queue_polling max_poll_messages
    # Khởi tạo Logger để lưu log cho mỗi phiên làm việc
    @logger = Logger.new "log/amazon_aws.log"
  end

  # Xong phần khởi tạo service. Chúng ta bắt tay vào phần xử lý
  # SQS message bằng QueuePoller nhé
  def execute
    # Khai báo các email status. Email có status nào sẽ được lưu vào
    # biến này để xử lý sau.
    email_statuses = {
      Delivery: [],
      Bounce: [],
      Complaint: []
    }

    # Thực hiện lấy message bằng polling
    @poller.poll do |queue_message|
      begin
        # Queue message body
        message_body = queue_body_parse queue_message.try(:body)
        # Notification message
        notification_message = queue_body_parse message_body[:Message]
        # Nhóm các email vào các status tương ứng để sử dụng lệnh
        # update_all để tăng performance thay vì mỗi message lại
        # update vào database
        email_statuses[notification_message[:notificationType].to_sym] <<
          notification_message[:mail].try(:[], "destination")
      rescue StandardError => execute_exception
        # Có lỗi với message đang lấy được. Lưu log và ném ra ngoại lệ
        # :skip_delete để giữ message lại cho lần xử lý sau
        log "Execute exception: #{execute_exception}", :error
        throw :skip_delete
      end
    end

    # Xử lý lại danh sách các email status
    email_statuses = improve_email_statuses email_statuses
    # Thực hiện update email status
    update_user_email_status email_statuses
  end

  # Các private method khác
  # Thông tin chứng chỉ Amazon AWS cho dịch vụ SQS.
  def amazon_credential_configs
    {
      region: <Queue region>,
      raise_response_errors: false,
      access_key_id: <Amazon Access key>,
      secret_access_key: <Amazon Secret Access key>
    }
  end

  # Dừng polling khi đạt được số lượng message mong muốn để tránh
  # chồng chéo giữa các phiên làm việc với nhau
  # Params:
  # +max_poll_messages+: Số lượng message tối đa muốn xử lý.
  def stop_queue_polling max_poll_messages
    @poller.before_request do |stats|
      throw :stop_polling if stats.received_message_count >= max_poll_messages
    end
  end

  # Ghi log
  # Params:
  # +log_content+: Nội dung log
  # +log_type+: Kiểu log: info, warn, error, debug
  def log log_content, log_type = :info
    log_content = "Amazon SQS Service: #{log_content}"

    @logger.send(
      log_type,
      log_content
    ) if ALLOWED_LOG_TYPES.include? log_type
  end

  # Parse SQS queue message thành JSON object
  # Params:
  # +queue_body+: SQS queue message body
  def queue_body_parse queue_body
    queue_body_obj = JSON.parse queue_body
    queue_body_obj.symbolize_keys
  rescue JSON::ParserError => json_parser_error
    raise "Error parse queue message body: #{json_parser_error}"
  end

  # Xử lý lại danh sách các email status bằng cách xóa bỏ các
  # giá trị rỗng (nếu có) và chuyển đổi mảng đa chiều thành mảng 1 chiều
  # Params:
  # +email_statuses+: Danh sách email status
  def improve_email_statuses email_statuses
    EMAIL_STATUSES.each do |email_status|
      next unless email_statuses[email_status].any?

      email_statuses[email_status].flatten!.compact!
    end

    email_statuses
  end

  # Update user email status theo Amazon SES status
  # Params:
  # +email_statuses: Danh sách email status
  def update_user_email_status email_statuses
    # Duyệt các email status mà chúng ta muốn xử lý
    EMAIL_STATUSES.each do |email_status|
      # Lấy các email theo status
      user_emails = email_statuses[email_status]
      # Nếu email status là Delivery thì update true, còn không
      # sẽ là false
      status = (email_status == :Delivery)

      # Bỏ qua nếu không có email nào cần phải xử lý
      next if user_emails.empty?

      # Tìm kiếm và update email status
      users = User.where email: user_emails
      update_count = users.update_all email_status: status

      # Nội dung log với các số lượng email tương ứng với các status
      # mà chúng ta đã xử lý được.
      log_content = <<-LOG_CONTENT
        \r
        ===============#{Time.zone.now}===============
          - SNS status: #{email_status.inspect}
          - Email number: #{user_emails.count}
          - Updated number: #{update_count}
        =====================================================
      LOG_CONTENT

      # Ghi log :D
      log log_content, :info
    end
  end
end
```

Vậy là đã xong phần service xử lý email status. Trong trường hợp bạn muốn test xem service làm việc ra sao, bạn hãy thêm method này vào service để tạo dummy data nhé:

```ruby
  # Gửi email tới Amazon SES mailbox simulator để tạo dummy data
  # Params:
  # +queue_number+: Số lượng dummy data mà bạn muốn có
  def make_dummy_queue queue_number = 5
    # Địa chỉ email tương ứng với status
    email_addresses = %w(bounce complaint success)

    (1..queue_number).each do |index|
      # Lấy ngẫu nhiên 1 email trong danh sách địa chỉ email
      email = email_addresses.sample
      to = "#{email}@simulator.amazonses.com"
      subject = "Subject #{index} to #{email.capitalize}"
      body = "Body #{index} to #{email.capitalize}"
      mail_content = {
        destination: {
          to_addresses: [to].flatten
        },
        message: {
          body: {
            html: {
              charset: "UTF-8",
              data: body
            }
          },
          subject: {
            charset: CHARSET,
            data: subject
          }
        },
        source: "#{sender_name} <#{sender_email}>"
      }

      email_status = AMAZON_SES.send_email mail_content
      log_content = "Send status: #{email_status.successful?}\n"

      if email_status.successful?
        log_content << "MessageID: #{email_status.message_id}"
      end

      log log_content, :debug
    end
  end
```

Khi nào bạn muốn tạo dummy data. Chỉ cần vào Rails Console thực hiện các lệnh sau :D

```
AmazonSqsService.new.make_dummy_queue <Số queue bạn muốn tạo>
```

OK, bây giờ mình hướng dẫn bạn add [**ResQue**](https://github.com/resque/resque) và [**Resque Scheduler**](https://github.com/resque/resque-scheduler) để tạo cron job chạy service của mình nhé :D!

Đầu tiên, thêm 2 gem vào `Gemfile`

```ruby
# Gemfile
gem "resque"
gem "resque-scheduler"
```

Chạy lệnh `bundle install` để cài gem. Tạo file initialize chứa cài đặt cho Resque

```ruby
# config/initializers/resque.rb

require "resque/scheduler"

redis_configs = {
  "host" => <Redis hostname>,
  "port" => <Redis port>,
  "db" => <DB number>,
  "options" => {
    "namespace" => <Redis namespace>
  }
}
Resque.redis = Redis.new redis_configs
Resque.redis.namespace = "resque:<SchedulerName>"

# If you want to be able to dynamically change the schedule,
# uncomment this line.  A dynamic schedule can be updated via the
# Resque::Scheduler.set_schedule (and remove_schedule) methods.
# When dynamic is set to true, the scheduler process looks for
# schedule changes and applies them on the fly.
# Note: This feature is only available in >=2.0.0.
# Resque::Scheduler.dynamic = true

Dir["#{Rails.root}/app/jobs/*.rb"].each{|file| require file}

# The schedule doesn't need to be stored in a YAML, it just needs to
# be a hash.  YAML is usually the easiest.
Resque.schedule = YAML.load_file(
  Rails.root.join("config", "resque_schedule.yml")
)
```

Tạo Resque Scheduler job:

```ruby
# app/jobs/update_user_email_status.rb

module UpdateUserEmailStatus
  @queue = :update_user_email_status

  def self.perform
    AmazonSqsService.new.execute
  end
end
```

Tạo file config để chạy các job:

```ruby
# config/resque_schedule.yml

UpdateUserEmailStatus:
  cron: "*/2 * * * *"
  description: Update user email status by SES notification type
```

Khi muốn test. Bạn nên để thời gian ngắn để có thể test được luôn. Trong phần cron kia là 2 phút sẽ chạy 1 lần. Các bạn có thể thay đổi theo ý mình tại trang [**này**](https://crontab.guru) nhé :D!

Để bắt đầu, chúng ta sẽ chạy 2 lệnh sau trên 2 tab terminal:

```
bundle exec rake resque:work
bundle exec rake resque:scheduler
```

OK, vậy là xong. Đến đây, series bài viết về Amazon SES và các dịch vụ khác để thực hiện việc xử lý (tracking) email status đã kết thúc. Nếu trong quá trình thực hiện, có phát sinh vấn đề về kỹ thuật hay muốn trao đổi thêm về Amazon SES, Amazon SNS, Amazon SQS thì mọi người để lại comment hoặc liên hệ với mình qua email nhé :D!

Chào thân ái và quyết thắng (chào)!
