---
date: 2016-01-31 02:07:00 +0700
title: PsySH - Interactive debugger and REPL for PHP
---

### Lời mở đầu

Bạn là một PHP programmer? Bạn đã từng phát triển website bằng một trong các framework hay CMS (Content Management System) như: Cake, Drupal, eZ Publish, Laravel, Magento, Patheon, Symfony, WordPress hay Zend?<!--more--> Nếu vậy, chắc hẳn ít nhiều bạn cũng biết đến chức năng tương tác với framework thông qua console (như terminal của Ubuntu hay CMD của Windows)? Vâng, mình thì chỉ biết đến, quan tâm và sử dụng chức năng hay ho này khi mình làm việc với Laravel framework là **Tinker**. Khoảng thời gian đầu, mình cũng không để ý đến chức năng này. Nhưng sau khi xem qua một video giới thiệu về Tinker của Laravel để debug ứng dụng thì mình đã bắt đầu chú ý hơn về nó. Và mình cũng chỉ biết dùng nó ở mức cơ bản là debug Model hay các service mình viết. Không hề quan tâm đến dòng chữ đầu tiên mỗi khi chạy lệnh Tinker là:

```
Psy Shell v0.6.1 (PHP 5.5.27-1+deb.sury.org~trusty+1 — cli) by Justin Hileman
```

Mình vẫn thầm nghĩ tại sao Laravel lại có thể phát triển được một công cụ hay ho như thế. Cho đến khi mình để ý dòng text ở trên :))! Mình liền Google **Psy Shell** thì được dẫn đến [trang chủ](http://psysh.org/) của nó. Sau đây, mình xin phép giới thiệu về nó - Psy Shell nhé.

### Psy Shell

**Psy Shell** là một "runtime developer console", nó hỗ trợ chúng ta "interactive debugger and REPL (Read-Eval-Print-Loop)" cho PHP. Nếu bạn đã hoặc thường xuyên debug code JS qua JavaScript console của trình duyệt hay thi thoảng bạn chạy thử đoạn code PHP với PHP interactive shell qua lệnh:

```
php -a
```

Thì chắc bạn cũng đã hiểu được phần nào sự tiện lợi của việc debug ứng dụng trực tiếp thông qua giao diện console. Okie, bây giờ mình sẽ giới thiệu về cách cài đặt Psy Shell, các câu lệnh chi tiết của nó và cách sử dụng nó trong source code của mình (nó không thuộc danh sách các framework hay CMS mà mình đã nêu ở trên) nhé!

#### Cài đặt Psy Shell

Cài đặt Psy Shell, bạn có 02 cách là thông qua **Composer** bằng lệnh:

```
composer g require psy/psysh:@stable
```

Sau đó chạy nó bằng lệnh:

```
~/.composer/vendor/psy/psysh/bin/psysh
```

Hoặc bạn cũng có thể download trực tiếp thông qua lệnh:

```
wget psysh.org/psysh
chmod +x psysh
./psysh
```

#### Các câu lệnh

Để xem danh sách các lệnh của Psy Shell, bạn có thể nhập `help`. Và để xem hướng dẫn chi tiết cũng như là các tham số cần thiết của một lệnh nào đó, bạn sử dụng lệnh `help <command_name>`. Mình sẽ đi chi tiết từng lệnh và ví dụ của nó nhé. Trong các ví dụ dưới đây, mình sẽ làm việc với Laravel nhé :D! Lệnh trong cặp `()` sau lệnh mình giới thiệu sẽ tương tự là alias của lệnh đó nhé. Ví dụ `help (?)` thì có nghĩa bạn sẽ sử dụng được `?` thay cho `help`.

* `help` (`?`): Hiện danh sách lệnh. Sử dụng `help <command_name>` để có thông tin chi tiết về lệnh đó
* `ls` (`list`, `dir`): Hiển thị danh các biến, hằng, hàm hay thể hiện (instance) của một class nào đó. Mình thử lấy các function của Request instance trong Laravel nhé. Các bạn có thể lấy ra nhiều thứ hơn với việc sử dụng lệnh `help <command_name>`

```
$request = app()->make('request');
ls -lm $request
```

* `dump`: Dump một object hay một đối tượng nào đó. Tương tự với `var_dump()` hay `print_r()` của PHP. Nhưng nó sử dụng **Var Dumpper** của **Symfony** để format.

```
dump($request->files)
```

* `doc` (`man`, `rtfm`): Hiển thị documentation của một variable, object, class, method, property hay constant nào đó. Với điều kiện người viết phải viết doc cho nó. Mình thử xem hàm `get()` của Request xem sao.

```
doc $request->get
```

* `show`: Hiển thị code của variable, object, clsss, method, property hay constance nào đó. Tiếp tục với ví dụ hàm `get()`:

```
show $request->get
```

* `wtf` (`last-exception`, `wtf?`): Hiển thị exception cuối cùng bị xảy ra.
* `whereami`: Cho bạn biết mình đang ở đâu :))!
* `throw-up`: Hiển thị exception cuối cùng và thoát khỏi Psy Shell
* `trace`: Hiển thị call stack hiện tại.
* `buffer` (`buf`): Hiển thị hoặc xóa các nội dung đệm của code mà bạn đã nhập. Trong trường hợp này, nếu bạn sử dụng `buf` thì bạn sẽ không còn thấy biến `$request` mà chúng ta đã làm từ ví dụ trước nữa :D!
* `clear`: Xóa nội dung trên màn hình hiện tại. Bạn cũng có thể sử dụng Ctrl+L để thực hiện việc này!
* `history` (`his`): Hiển thị danh sách các lệnh mà bạn đã sử dụng từ trước tới giờ (nếu có).
* `exit` (`quit`, `q`): Thoát khỏi Psy Shell.

Ngoài ra, Psy Shell cũng cho phép chúng ta cài đặt nó như hạn chế số dòng của history, cho phép lưu lại các câu lệnh giống nhau vào history hay kích thước của tập tin chứa các câu lệnh đã sử dụng, ... bằng cách bạn tạo một file với tên `config.php` (nếu chưa có) tại thư mục `~/.config/psysh/`. Để có chi tiết, bạn có thể đọc [tại đây](http://psysh.org/#configure) nhé. Bây giờ chúng ta thử cài đặt Psy Shell cho một ứng dụng PHP khác ngoài danh sách những framework hay CMS mình đã kể trên nhé. Cái quan trọng là chúng ta cần phải biết cách lấy khung của cả framework hay CMS. Mình chọn [CodeIgniter](https://codeigniter.com/) để làm ví dụ (vì đây là framework đầu tiên mình làm việc, nên ít nhiều vẫn còn rất nhiều cảm tình với nó =D).

Đầu tiên, chúng ta sẽ tải source code của nó về tại [đây](https://github.com/bcit-ci/CodeIgniter/archive/2.2.6.zip). Mình chọn v2x nhé. Xả nén file zip đã tải về, sau đó đi vào thư mục CI mà bạn vừa xả nén, cài đặt Psy SH bằng Composer:

```
composer require psy/psysh
```

Tạo một file với tên tùy ý. Mình chọn là ``ci-shell`` nhé :))! Trong CI, bạn có thể lấy super object (chứa khung của CI framework) bằng function ``get_instance()``. Và để có thể gọi được function này, bạn mở file ``index.php`` của CI ra sẽ thấy, nó khai báo một loạt các constant cần thiết, sau đó require file ``CodeIgniter.php`` trong thư mục ``core`` của system. Nếu chúng ta chỉ require mình ``CodeIgniter.php`` trong file ``ci-shell`` không thôi thì chương trình sẽ không chạy. Vì CI luôn kiểm tra xem có constant ``BASE_PATH`` không. Vậy, để tránh viết lại cả file ``index.php`` vào ``ci-shell``, chúng ta sẽ thực hiện include luôn cả index.php vào ci-shell của chúng ta nhé:

```php
<?php

ob_start();

require_once __DIR__ . '/index.php';
require_once __DIR__ . '/vendor/autoload.php';

use Psy\Shell;

ob_end_clean();

Shell::debug(get_defined_vars(), get_instance());
```

OK, giờ bạn có thể sử dụng Psy Shell với CI bằng lệnh:

```
php ci-shell
```

Bây giờ bạn có thể tương tác với CI rồi. Bạn thử xem nhé ^^!

![ci-shell-demo.png](/assets/images/posts/8cbad496-3c4f-4321-aba7-1561b82e6109.png)
