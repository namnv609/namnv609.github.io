---
date: 2017-01-27 03:39:00 +0700
title: Node.JS Yargs - Build interactive command line tools
---

Sau loạt bài viết về [**Amazon SES, SNS and SQS**](/posts/amazon-ses-sns-and-sqs.html) khá là khô khan (vì khó để thực hiện việc kiểm thử) và không có tính ứng dụng rộng rãi.<!--more--> Mình sẽ quay lại với chủ đề có tính thực tiễn cao hơn. Hôm nay mình xin chia sẻ về một Node.JS module hỗ trợ chúng ta trong việc parse các tham số cho ứng dụng console. Như mình đã từng giới thiệu một module cho Python là [**Argparse**](https://viblo.asia/p/pymotm-argparse-l5XRBVeVRqPe), hôm nay mình sẽ giới thiệu module này cho ngôn ngữ Node.JS, đó là [**Yargs**](http://yargs.js.org/), một Node.JS module cho việc parse các tham số trong ứng dụng console. OK, chúng ta cùng bắt đầu tìm hiểu nó nhé.

# What's Yargs
Yargs là một module giúp bạn xây dựng một ứng dụng console có tính tương tác cao và thân thiện với người sử dụng. Yargs sẽ mang lại cho bạn những gì?
* Các câu lệnh và (nhóm) các tùy chọn (ví dụ: `module run -n --force`)
* Tự động sinh ra menu trợ giúp dựa trên các tham số của ứng dụng
* Auto complete cho các câu lệnh và tùy chọn
* Và nhiều hơn thế...

Với những đặc tính đã kể trên, Yargs sẽ giúp bạn tập trung vào xây dựng và phát triển tính năng của ứng dụng mà không cần phải bận tâm đến cách đọc các tham số từ người dùng.

# Getting Started
Trước khi bắt đầu với Yargs, chúng ta thử nhìn một ứng dụng đơn giản bằng Node.JS mà không có Yargs nhé. Ứng dụng đơn giản này sẽ nhận tên và tuổi của người dùng đã nhập và hiển thị lên màn hình console nhé.

```javascript
// no-yargs.js
var args = process.argv;

console.log("Your name: %s\nYour age: %s years old", args[2], args[3]);
```

Chạy thử ứng dụng và xem kết quả nhận được nhé

```
$ node no-yargs.js name 24
Your name: name
Your age: 24 years old
```

Bạn nhìn vào đoạn code trên. Tại sao chúng ta lại lấy từ index số 2 và số 3 cho name và age? Bạn hãy thử log ra biến `args` để hiểu lý do tại sao nhé :D?

Ngoài việc đọc code khó hiểu thì còn bất tiện gì nữa khi không có Yargs nhỉ? Có lẽ là vị trí của các tham số mà người dùng sẽ nhập. Ví dụ: `node no-yargs.js 24 name` thì chúng ta sẽ thấy kết quả thật là buồn cười, đúng không :D? Bây giờ chúng ta thử viết lại ứng dụng trên bằng cách dùng module Yargs nhé.

Đầu tiên, bạn cần cài Yargs bằng lệnh

```
npm install yargs --save
```

Sau đó thực hiện việc viết lại ứng dụng:

```javascript
// with-yargs.js
var argv = require("yargs").argv;

console.log("Your name: %s\nYour age: %s years old", argv.name, argv.age);
```

Chạy thử ứng dụng xem sao nhé :D

```
$node with-yargs.js --name name --age 24
Your name: name
Your age: 24 years old
$ node with-yargs.js --age 24 --name name
Your name: name
Your age: 24 years old
```

Nhìn ứng dụng của chúng ta thân thiện chứ nhỉ? Người dùng sẽ biết cần phải nhập những gì, hay những tham số nào cho ra những dữ liệu nào và dù người dùng đổi vị trí của các tham số bạn cũng không phải lo sai dữ liệu đầu ra :D. Bây giờ chúng ta sẽ tìm hiểu chi tiết một số method của module Yargs này nhé :D

# Yargs methods
## `.alias(key, alias)`
Đặt `alias` cho tên của một tham số (`key`). Method có thể nhận vào 1 object chứa `key` và `alias` của key nếu bạn muốn khai báo nhiều. Chúng ta update lạ file `with-yargs.js` theo code dưới đây và thử nhé :D

```javascript
// with-yargs.js
var argv = require("yargs")
  .alias({
    'name': 'n',
    'age': 'a'
  })
  .argv;

console.log("Your name: %s\nYour age: %s years old", argv.name, argv.age);
```

Bạn chạy thử 2 cách dưới đây và xem kết quả nhé:

```
$node with-yargs.js --name name --age 24
$node with-yargs.js -n name -a 24
```

## `.argv`
Chuyển đổi các tham số thành object.

```javascript
// argv.js

var yargs = require("yargs");

console.log(yargs.argv);

/**
$ node argv.js -f foo -b bar
{ _: [], f: 'foo', b: 'bar', '$0': 'argv.js' }
*/
```

## `.array(key)`
Thông báo với parser rằng `key` này đang nhận giá trị là một array. Các giá trị được phân cách nhau bằng dấu space. Các bạn xem ví dụ để rõ hơn:

```javascript
// array.js
var argv = require("yargs")
  .array("id")
  .argv;

console.log(argv.id);
console.log("SELECT * FROM users WHERE id IN (%s)", argv.id.join(","));
```

Chạy thử:

```
$node array.js --id 1 2 3 4 5
[ 1, 2, 3, 4, 5 ]
SELECT * FROM users WHERE id IN (1,2,3,4,5)
```
Bạn thử comment đoạn `.array("id")` và chạy lại xem sao :v?

## `.boolean(key)`
Thông báo với parser rằng `key` này có kiểu dữ liệu là `boolean`. `key` boolean có giá trị mặc định là `false`. Nếu bạn muốn thay đổi giá trị này, hãy sử dụng method `default(key, value)`. Bạn có thể xem ví dụ bên dưới:

```javascript
// boolean.js
var argv = require("yargs")
  .boolean("production")
  .alias("prod", "production")
  .argv;

console.log(argv);

/**
$ node boolean.js --prod
{ _: [], production: true, prod: true, '$0': 'boolean.js' }
$ node boolean.js
{ _: [], production: false, prod: false, '$0': 'boolean.js' }
*/
```

## `.check(callbackFunc)`
Dùng để kiểm tra xem các tham số được truyền vào có hợp lệ hay không. Tham số của method `.check` là 1 function. Nếu function này ném ra 1 error, chương trình sẽ in ra lỗi đó, thông tin hướng dẫn và thoát.
_Từ phiên bản 5.0.0 trở đi. `callbackFunc` phải trả về 1 là return `true` hoặc `throw(new Error("message"))`. Nếu không, Yargs sẽ in nội dung của `callbackFunc`_

```javascript
// check.js
var argv = require("yargs")
  .check(function(argv) {
    if (typeof argv.id !== "number") {
      throw(new Error("ID must be a number"));
    }

    return true;
  })
  .argv;

console.log("ID: %d", argv.id);

/**
$ node check.js --id lorem
ID must be a number
$ node check.js --id 2
ID: 2
*/
```

## `.choices(key, choices)`
Giới hạn giá trị cho `key` bằng cách định nghĩa tập hợp (mảng) các giá trị `choices`. Nếu method này được gọi nhiều lần cho một key, tất cả các giá trị được liệu kê (`choices`) sẽ được gộp lại thành một.

```javascript
// choices.js

var argv = require("yargs")
  .choices("service", ["Gmail", "Hotmail", "Yahoo!Mail"])
  .alias("s", "service")
  .help()
  .argv;

console.log("Your choice service: %s", argv.service);

/**
$ node choices.js -s Gmail
Your choice service: Gmail
$ node choices.js -s Zoho
Options:
  --help  Show help                                                    [boolean]

Invalid values:
  Argument: service, Given: "Zoho", Choices: "Gmail", "Hotmail", "Yahoo!Mail"
*/
```

## `.coerce(key, callbackFunc)`
Tiền xử lý dữ liệu đầu vào do người dùng nhập. Tham số của `callbackFunc` là giá trị của `key`. `callbackFunc` cần trả về giá trị mới cho `key` hoặc ném ra một lỗi. `.coerce` cũng có thể nhận 1 object gồm object key là `key` muốn xử lý, object value là `callbackFunc` xử lý cho key đó. Chúng ta thử làm 1 ví dụ là nhận vào ngày tháng và năm rồi chuyển đổi giá trị đó thành miliseconds (tính từ 1970/01/01) nhé.

```javascript
// coerce.js
var argv = require("yargs")
  // .coerce("date", function(date) {
  //   return new Date(date).getTime();
  // })
  // .coerce(["date", "bod"], function(value) {
  //   return new Date(value).getTime();
  // })
  .coerce({
    "date": function(date) {
      return new Date(date).getTime();
    },
    // "bod": function(bod) {
    //   return new Date(bod).getTime();
    // }
  })
  .argv;

console.log("Miliseconds since 1979/01/01: %s", argv.date);
// console.log("Miliseconds since 1979/01/01 to your birthday: %s", argv.bod);

/**
$ node coerce.js --date 2012-01-01
Miliseconds since 1979/01/01: 1325376000000
*/
```

---

Bài viết đã khá là dài mà các methods của Yargs còn rất nhiều. Mọi người cũng đã có cái nhìn căn bản với module Yargs nên mình xin phép tạm dừng tại đây. Mình sẽ giới thiệu tiếp các method còn lại của Yargs trong các bài viết tới. Hẹn gặp lại mọi người trong phần tiếp theo của bài viết về **Node.JS Yargs - Build interactive command line tools** nhé :D!
