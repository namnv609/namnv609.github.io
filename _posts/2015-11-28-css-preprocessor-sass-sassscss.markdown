---
date: 2015-11-28 09:37:00 +0700
title: CSS Preprocessor - SASS (SASS&SCSS)
thumbnail: /assets/images/posts/351e1bee-4589-4556-9072-25f262d50392.png
---

Hôm nay, mình xin phép giới thiệu về SASS - một CSS Preprocessor khá mạnh và phổ biến. Trước khi đi vào tìm hiểu SASS, chúng ta sẽ tìm hiểu qua một số kiến thức căn bản về CSS, để biết về mục đích và lý do vì sao mình viết bài chia sẻ này nhé.<!--more-->

#### CSS là gì?

CSS là chữ viết tắt của cụm từ tiếng Anh **Cascading Style Sheet**. Nó là một ngôn ngữ định kiểu, quy định cách trình bày cho các tài liệu được viết bằng HTML (**H**yper**T**ext **M**arkup **L**anguage - Ngôn ngữ đánh dấu siêu văn bản) và XHTML (e**X**tensible **H**yper**T**ext **M**arkup **L**anguage - Ngôn ngữ đánh dấu siêu văn bản mở rộng). Các đặc tả kỹ thuật của nó được duy trì và phát triển bởi W3C (World Wide Web Consortium). Nếu so sánh vui về việc xây dựng giao diện (UI) của website là xây nhà, thì HTML là gạch (đảm nhiệm việc dựng kết cấu của ngôi nhà), còn CSS là sơn (tô điểm và trang trí cho ngôi nhà) :D

#### Quy tắc tải CSS của trình duyệt

Khi trang web được tải, browser sẽ tiến hành đọc toàn bộ style mà trang web có thể áp dụng bao gồm:

* Style mặc định của trình duyệt
* External style (style bên ngoài, được nhúng vào thông qua thẻ `<link />`)
* Internal style (style đặt trong cặp thẻ `<style></style>`)
* Inline style (style viết trực tiếp trong thẻ HTML thông qua thuộc tính (attribute) **style**

Sau đó, browser sẽ tổng hợp lại vào một bộ CSS ảo của nó. Và nếu có các thuộc tính giống nhau của một selector (class, tag, id, ...) thì browser sẽ ưu tiên áp dụng style nào được nạp sau cùng. Như vậy, thì thứ tự ưu tiên sẽ là:

```
Inline style > Internal style > External style > Browser default style
```

Vậy, có cách nào để thay đổi (chiếm) độ ưu tiên của các CSS property hay không? Câu trả lời là có. Nếu muốn thay đổi độ ưu tiên của một thuộc tính, bạn chỉ cần thêm `!important` vào sau giá trị của thuộc tính đó. Và thứ tự ưu tiên áp dụng của `!important` cũng giống như thứ tự áp dụng bên trên nhé => `!important` của Inline style sẽ là cao nhất :)) (vì vậy, mọi người nên tránh hoặc hạn chế sử dụng `!important` nhé)!

#### Quy tắc viết CSS

Khi viết CSS cho website, mọi người nên hạn chế viết inline style, internal style và sử dụng `!important` nhé. Nên viết external style (các style trong file `*.css` riêng biệt) để dễ dàng bảo trì, dễ dàng thay đổi thứ tự ưu tiên cho các thuộc tính của một số thẻ riêng biệt mà không cần phải sử dụng đến `!important`. Và để tránh chồng chéo các thuộc tính của CSS, chúng ta nên viết chi tiết cho các selector (class, tag, id, ...) nhé. Để dễ hình dung, mình sẽ có một ví dụ (không bao gồm sử dụng `!important` và inline style nhé). Viết style cho một class bằng internal style và external style như sau:

```html
<link type="text/css" rel="stylesheet" href="style.css" />
<style>
  .child {
    color: #00F; /*#00FFFF - Blue*/
  }
</style>
<div class="wrapper">
  <div class="first">
    <div class="second">
      <div class="third">
        <div class="child">Welcome</div>
      </div>
    </div>
  </div>
</div>
```

```css
.third .child {
  color: #F00; /*#FF0000 - Red*/
}

.second .third .child {
  color: #CCC; /*#CCCCCC - Silver*/
}

.first .second .third .child {
  color: #639; /*#663399 - Pure violet*/
}

.wrapper .first .second .third .child {
  color: #0FF; /*#00FFFF - Cyan*/
}
```

Nhìn vào đoạn hai đoạn code trên, bạn nghĩ văn bản trong class `.child` sẽ có màu gì? Theo thứ tự áp dụng bên trên, thì nó sẽ là màu #00F (blue) vì màu #00F cho class `.child` được viết trong cặp thẻ `<style></style>` - internal style mà. Nhưng màu của nó lại là màu cyan (#0FF) cơ :D! Sao lại thế (?)? Vì chúng ta viết màu #0FF cho class `.child` chi tiết hơn (lần lượt từ cái trên nhất xuống, nên nó được ưu tiên tương tự như inline style). Theo đó, nếu ta viết CSS thuần thì việc viết chi tiết như thế này sẽ rất nhàm chán, và sẽ phải lặp đi lặp lại một loạt các class cha giống nhau cho nhiều class con bên trong. Thảm họa đấy (với một thằng lười như mình :D). Vậy nên, chúng ta sẽ đi tìm hiểu SASS - một CSS Preprocessor nhóe!

#### CSS Preprocessor là gì?

CSS Preprocessor được hiểu là ngôn ngữ tiền xử lý CSS, hay hiểu nôm na nó là phiên bản mở rộng của ngôn ngữ CSS. Nó có nhiệm vụ giúp bạn logic hóa và cấu trúc các đoạn mã CSS để cho CSS nó đến gần hơn với một ngôn ngữ lập trình. Ngoài ra, nó còn có một số lợi ích sau:
* Tiết kiệm thời gian viết CSS
* Dễ dàng bảo trì và phát triển
* Tính linh hoạt và tái sử dụng
* Các tập tin CSS được tổ chức một cách rõ ràng

Có khá nhiều CSS Preprocessor như [SASS](http://sass-lang.com/), [LESS](http://lesscss.org/), [Stylus](http://learnboost.github.io/stylus/), ... Và CSS preprocessor đầu tiên mình biết và sử dụng là LESS :D! Nhưng mình không sử dụng nó nhiều, cho đến khi biết đến SASS. Vậy nên mình sẽ giới thiệu cái mà mình sử dụng nhiều nhất (từ project đầu tiên cho đến project hiện tại) là SASS nhé ^^! Cùng mình đi tìm hiểu xem SASS là gì nào.

#### SASS là gì?

SASS (**Syntactically Awesome StyleSheets**) là một phần mở rộng của CSS, nó giúp chúng ta sử dụng biến (variables), quy tắc xếp chồng (nested rules), mixins, thừa kế (selector inheritance), hàm (functions), ... và hoàn toàn tương thích với cú pháp của CSS.

#### Các đặc tính của SASS

* Hoàn toàn tương thích với CSS
* Mở rộng ngôn ngữ như các biến (variable), mixins, hàm (function), ...
* Nhiều function hữu ích cho các thao tác với màu sắc và các giá trị khác
* Các đặc tính nâng cao như các control directive
* Có cấu trúc, tùy biến đầu ra

SASS có hai định dạng file là `*.sass` và `*.scss`. Và cách viết của hai định dạng này cũng là khác nhau (nhưng các control directive, function thì có cùng một ý nghĩa). Và nếu bạn biết một cái, thì bạn cũng có thể viết được cái thứ hai. Điểm qua một số sự khác biệt trong cách viết (mà mình biết) của hai định dạng file này nhé:
* `*.sass`:
  * Sử dụng indent để thể hiện quy tắc xếp chồng (nested rules)
    * Không cần sử dụng `;` khi kết thúc một property
    * Khai báo mixins bằng ký tự `=`
    * Sử dụng mixins bằng ký tự `+`
* `*.scss`:
  * Sử dụng dấu `{` và `}` để thể hiện quy tắc xếp chồng (nested rules)
    * Sử dụng `;` để kết thúc một property
    * Khai báo mixins bằng directive `@mixin`
    * Sử dụng mixins bằng directive `@include`

Ví dụ về hai cách viết trên:

```sass
$vendorPrefixes: (-webkit-, -moz-, -khtml-, -o-, -ms-)

=css3-prefix($property, $value...)
  @each $vendorPrefix in $vendorPrefixes
    #{$vendorPrefix}#{$property}: unquote($value)
  #{$property}: unquote($value)

body
  .box
    +css3-prefix(border-radius, 3px)
```

```scss
$vendorPrefixes: (-webkit-, -moz-, -khtml-, -o-, -ms-);

@mixin css3-prefix($property, $value...) {
  @each $vendorPrefix in $vendorPrefixes {
    #{$vendorPrefix}#{$property}: unquote($value);
  }
  #{$property}: unquote($value);
}

body {
  .box {
    @include css3-prefix(border-radius, 4px);
  }
}
```

Bây giờ, mình sẽ giới thiệu một số control directive căn bản mà mình hay dùng và ví dụ cho các từng directive đó nhé.

#### @-Rules và Directives

##### __`@import`__

Cho phép bạn import các rule, style, biến, mixins, functions, ... từ một file SASS khác. Và nó sẽ được gộp lại thành một file khi xuất ra file CSS. Directive `@import` nhận một chuỗi là tên file sẽ được import. Mặc định, khi tên file không có phần mở rộng (extension), thì nó sẽ ưu tiên tìm file có phần mở rộng là `*.scss` và `*.sass`!

```sass
@import "../common/css3-mixins"
// Hoặc
@import "../common/css3-mixins.sass"
```

Hoặc bạn cũng có thể import nhiều file cùng một lệnh:

```sass
@import "../common/css3-mixins", "../components/_header"
```

##### __`@extend`__

Cho phép bạn thừa kế các property của một class khác.

```sass
.alert
  padding: 10px
  font:
    family: tahoma
    size: 12px
.error
  @extend .alert
  color: #F00
```

```css
.alert, .error {
  padding: 10px;
  font-family: tahoma;
  font-size: 12px;
}

.error {
  color: #F00;
}

```

##### __`@each`__

Giúp bạn duyệt một danh sách (list) hay một map các giá trị. Dùng trong trường hợp phải viết một số lệnh giống nhau, nhưng chỉ khác chút về giá trị của property.

```sass
$userStatuses: (online: #0F0, idle: #FF0, offline: #CCC)

@each $class, $color in $userStatuses
  .user-#{$class}
    background-color: #{$color}
```

```css
.user-online {
  background-color: #0F0;
}

.user-idle {
  background-color: #FF0;
}

.user-offline {
  background-color: #CCC;
}
```

##### __`@mixin`__

Giúp bạn định nghĩa một khối các style có thể được sử dụng lại nhiều lần. Trong SASS, ngoài `@mixin` còn `@function`, về bản chất nó giống nhau. Nhưng khác một chỗ, `@mixin` không trả về (`@return`) giá trị nào cả (gọi nó là void function cũng được :D), còn `@function` thì luôn phải trả về một giá trị. Về ví dụ, bạn có thể xem lại phần giới thiệu về cách viết giữa `*.sass` và `*.scss` ở trên nhé :D!

##### __`@include`__

Dùng để gọi các `@mixin` (trong `*.scss`). Ví dụ, bạn cũng có thể xem ở phần giới thiệu về cách viết của `*.sass` và `*.scss`!

##### __`@content`__

Directive này giúp bạn lấy toàn bộ nội dung của một khối để đưa vào `@mixin`. Mình sẽ viết ví dụ để mọi người dễ hình dung.

```sass
=keyframes($animationName)
  @-webkit-keyframes $animationName
    @content
  @-moz-keyframes $animationName
    @content
  @-o-keyframes $animationName
    @content
  @keyframes $animationName
    @content

+keyframes(fadeIn)
  0%
    opacity: 0
  50%
    opacity: 0.5
  100%
    opacity: 1
```

```css
@-webkit-keyframes fadeIn {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}
@-moz-keyframes fadeIn {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}
@-o-keyframes fadeIn {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}
@keyframes fadeIn {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}
```

##### Nested properties

```sass
body
  font:
    family: Tahoma, verdana, sans-serif
    size: 12px
    weight: bold
  background:
    image: url(path/to/image)
    repeat: no-repeat
    position: center top
```

```css
body {
  font-family: Tahoma, verdana, sans-serif;
  font-size: 12px;
  font-weight: bold;
  background-image: url(path/to/image);
  background-repeat: no-repeat;
  background-position: center top;
}
```

Trên thực tế, trong trường hợp viết font và background như ví dụ trên là không cần thiết, vì bạn có thể gộp nó lại thành 1 property thôi là đủ. Nhưng mình vẫn viết để mọi người hiểu về nested properties thôi :D

```sass
body
  font: bold 12px tahoma, verdana, sans-serif
    background: url(path/to/image) no-repeat center top
```

##### Parent selector

```sass
a
  color: #FFF
  text-decoration: none
  &:hover
    text-decoration: underline
```

```css
a {
  color: #FFF;
  text-decoration: none;
}
a:hover {
  text-decoration: underline;
}
```

Hoặc:

```sass
.box
  border: 1px solid #000
  padding: 5px
  &-radius
    @extend .box
    border-radius: 4px
  &-shadow
    @extend .box
    box-shadow: 10px 10px 5px #888
```

```css
.box, .box-radius, .box-shadow {
  border: 1px solid #000;
  padding: 5px;
}
.box-radius {
  border-radius: 4px;
}
.box-shadow {
  box-shadow: 10px 10px 5px #888;
}
```

Bài viết của mình đến đây là kết thúc. Để hiểu rõ và nhiều hơn. Mọi người có thể vào trang chủ của [SASS](http://sass-lang.com) để xem thêm chi tiết về các directive và function nhé ^^!

_Chia sẻ thêm về quy tắc gộp mã màu hex trong CSS._

Mã màu hex trong CSS bao gồm 03 cặp hex. Nếu các cặp đó giống nhau hoàn toàn và đồng nhất giữa 03 cặp, bạn có thể gộp lại thành 01 ký tự cho gọn. Ví dụ:
* Màu đỏ: #FF0000 -> Chúng ta có 03 cặp FF, 00 và 00. Theo quy tắc trên, ta gộp lại sẽ còn F, 0 và 0 -> #F00. Tương tự với các màu khác như trắng (#FFFFFF -> #FFF), xanh trời (#0000FF -> #00F), ...
* Với trường hợp các màu sắc khác không trùng nhau và đồng nhất giữa 03 cặp hex, ví dụ là màu xanh lá (#008000) thì lại không gộp được, vì nó chỉ có 02 cặp trùng nhau: 00 -> 0, 80 -> ???, 00 -> 0 => ta vẫn phải dùng đủ 06 ký tự hex của nó :D!

// Hy vọng mọi người hiểu mình đang nói gì (lay2)

Bài viết của mình có hơi lủng củng và sử dụng quan điểm cá nhân, mong mọi người thông cảm (yaoming)!
