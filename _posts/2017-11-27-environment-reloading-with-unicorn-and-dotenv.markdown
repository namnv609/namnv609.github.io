---
date: 2017-11-27 21:14:00 +0700
title: Environment reloading with Unicorn and Dotenv
---

Trước đây mình đã giới thiệu với mọi người về auto deployment với ứng dụng **Ruby on Rails** thông qua Capistrano với tiêu đề [Zero downtime deployment for Rails with Capistrano and Unicorn](/posts/zero-downtime-deployment-for-rails-with-capistrano-and-unicorn.html) và mình đã gặp một vấn đề trong thực tế muốn chia sẻ với mọi người với hy vọng mọi người sẽ không mắc phải và có kinh nghiệm xử lý. Đó là việc nạp lại các thông tin cài đặt từ file `.env` với Unicorn.<!--more-->

Số là mình sử dụng gem [`dotenv-rails`](https://github.com/bkeepers/dotenv) để lưu các cài đặt trong ứng dụng. Và trong quá trình vận hành, một số cài đặt cần phải thay đổi giá trị và khởi động lại ứng dụng để nạp lại các thay đổi đó (trên lý thuyết trong bài viết trước mình đã giới thiệu). Với việc zero downtime, tức là không có thời gian trễ (ngừng phục vụ) trong khoảng thời gian chúng ta khởi động lại ứng dụng bằng cách sử dụng signal `USR2` trong khi kill Unicorn process (bạn có thể tìm hiểu thêm các signal của Unicorn [tại đây](https://bogomips.org/unicorn/SIGNALS.html)). Nhưng trên thực tế mình đã gặp phải và phải loay hoay tìm hiểu và xử lý, đó là nó không hề nhận các giá trị đã thay đổi (nhưng với việc thêm mới một cài đặt khác thì OK) với việc restart Unicorn mà cần phải stop Unicorn process xong start lại (sẽ gây ra việc delay và không còn là zero downtime nữa) nó mới nhận. Và bài viết này là để chia sẻ với mọi người về cách xử lý trong trường hợp này. Để dễ hình dung, chúng ta cùng thử tái hiện lại để xem sự thức có đúng như thế không bằng cách tạo một server để deploy (sử dụng Docker) ứng dụng, thay đổi một vài giá trị đã có sang giá trị mới và khởi động lại xem sao và so sánh nó sau khi đã khắc phục nhé.

Trước tiên, chúng ta sẽ chuẩn bị một server theo các hướng dẫn từ bài viết [này](/posts/zero-downtime-deployment-for-rails-with-capistrano-and-unicorn.html). Sau đó, phần [**cài đặt source code**](/posts/zero-downtime-deployment-for-rails-with-capistrano-and-unicorn.html) thì bạn thêm cho mình một số bước sau:

1. `Gemfile`
    Bạn thêm gem `dotenv-rails` vào `Gemfile` để có thể sử dụng `.env` file
2. `config/deploy.rb`
    Thêm `.env` vào phần `linked_files`

    ```ruby
    set :linked_files, fetch(:linked_files, []).push("config/database.yml", "config/secrets.yml", ".env")
    ```
3. Thêm `.env` trong thư mục `/var/www/html/#{fetch(:application)}/shared/` trên server (Docker container)

Vậy là tạm thời đã xong, chúng ta bắt đầu thử nghiệm nhé. Đầu tiên là tạo một controller, mình sẽ chọn tên là `homes_controller` và cho nó là root như sau:

```ruby
# app/controllers/homes_controller.rb
class HomesController < ApplicationController
  def index
    @price = 28000
    @vat = ENV["TAX_PERCENTAGE"]
    @total = @price + get_vat
  end

  private
  def get_vat
    @price.to_f * (@vat.to_f / 100.0)
  end
end
```

```Ruby
# config/routes.rb
Rails.application.routes.draw do
  root "homes#index"
end
```

```
# .env
TAX_PERCENTAGE=10
```

Tiếp theo, đến phần view:

```ruby
# app/views/homes/index.html.erb
Price<sup>1</sup>: <%= @price %>$<br />
VAT<sup>2</sup>: <%= @vat %>%<br />
Total (1 + 2): <%= @total.to_i %>$
```

Xong, bây giờ, bạn commit rồi deploy lên server và chạy thử ứng dụng. Bạn sẽ nhận được một view như sau:

```
Price1: 28000$
VAT2: 10%
Total (1 + 2): 30800$
```

Trên server, sửa lại giá trị `TAX_PERCENTAGE` sang `8` và khởi động lại Unicorn process bằng 1 trong 2 cách sau:

1. Trên server: `ps aux | grep -i "unicorn master" | awk 'NR==1{print $2}' | xargs kill -USR2`
2. Ở client: chạy lệnh `bundle exec cap <stage> unicorn:restart`

Sau khi đã khởi động xong, thử refresh lại trang xem kết quả. OMG, nó chẳng thay đổi gì cả :/! Giờ chúng ta sẽ stop và start lại Unicorn process bằng 2 lệnh sau trên client:

1. `bundle exec cap <stage> unicorn:stop`
2. `bundle exec cap <stage> unicorn:start`

Refresh lại trang và xem kết quả. Haiz, ơn Giời là nó đã thay đổi. Nhưng những người dùng khác truy cập vào trang trong thời điểm mình stop và start lại ứng dụng sẽ bị ăn lỗi 502 và nó không còn là zero downtime nữa.

OK, giờ chúng ta sẽ đi xử lý nhé. Công việc rất đơn giản nhưng quan trọng, là chúng ta sẽ thêm 2 dòng lệnh vào file cài đặt Unicorn tương ứng với môi trường mà chúng ta sử dụng (staging, production, ...).

```ruby
# config/unicorn/<stage>.rb
require "dotenv"
# ...
before_exec do |_|
  #...
  ENV.update Dotenv::Environment.new ".env"
  #...
end
# ...
```

Đối với Puma (hiện tại mình vẫn chưa test thử với Puma), bạn có thể làm như sau:

```ruby
# config/puma.rb
require "dotenv"
# ...
on_worker_bot do
  # ...
  ENV.update Dotenv::Environment.new ".env"
  # ...
end
# ...
```

Xong, bây giờ bạn commit và push code lên Github rồi thực hiện deploy lại. Nhưng bạn vẫn cần phải stop và start lại ứng dụng (không phải restart) một lần nữa để mọi thứ có thể làm việc được chính xác. Sau khi xong, từ bây giờ, bạn có thể thoải mái thay đổi giá trị của `.env` và restart lại ứng dụng mà không gặp trở ngại nào liên quan đến việc cache nữa. Bạn có thể xem video demo:

<iframe width="560" height="315" src="https://www.youtube.com/embed/S6QyJNSIPH8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Bài viết của mình đến đây là kết thúc. Có thể nó ngắn, nhưng nó là kinh nghiệm của mình muốn chia sẻ với mọi người để mong mọi người không gặp phải trường hợp như mình. Hẹn gặp lại mọi người trong bài viết sau.

* Link Github project: [https://github.com/namnv609/rails-environment-loading](https://github.com/namnv609/rails-environment-loading)
* Link tham khảo: [http://sorentwo.com/2014/08/27/environment-reloading.html](http://sorentwo.com/2014/08/27/environment-reloading.html)
