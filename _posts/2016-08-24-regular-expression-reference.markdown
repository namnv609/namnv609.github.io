---
date: 2016-08-24 13:35:00 +0700
title: Regular Expression Reference
---

Tạm dừng series bài viết về module của Python (**PyMOTM**) để tránh nhàm chán, hôm nay mình xin chia sẻ về **Regular Expression** (hay còn gọi là biểu thức chính quy) trong lập trình. Bài viết này mình sẽ giới thiệu về Regular Expression, những tokens, modifiers mà RegEx hỗ trợ.<!--more--> Mình sẽ dịch nó (theo những gì mình hiểu) kèm với những ví dụ đi cùng (nếu có thể) để bạn có thể hiểu hơn về RegEx nhé.

### Regular Expression là gì?

Regular Expression (Biểu thức quy tắc - gọi tắt là RegEx) là một chuỗi ký tự đặc biệt được dùng làm mẫu (pattern) để phân tích, so sánh sự trùng khớp của một tập hợp các chuỗi cho trước nào đó. Vậy trong lập trình, RegEx dùng để làm gì? Chúng ta thường xuyên sử dụng nó để kiểm tra tính hợp lệ (validate) của form (dữ liệu do người dùng nhập và đưa lên server cho chúng ta xử lý). Hoặc nó cũng được sử dụng để tìm kiếm, thay thế, xử lý hay trích xuất dữ liệu từ một tập hợp các đoạn văn bản nào đó.

Vâng, vậy là chúng ta đã có cái nhìn cơ bản về RegEx rồi. Giờ chúng ta sẽ đi tìm hiểu các tokens và modifiers của RegEx nhé. Mình sẽ chia bài viết thành nhiều phần. Mỗi phần là một tập hợp các tokens được sử dụng để xử lý trong các trường hợp yêu cầu cụ thể.

### RegEx tokens

#### General tokens

 * `\n`: Newline. Tìm kiếm ký tự xuống dòng (line-feed)
 * `\r`: Carriage return. Tìm kiếm ký tự điều khiển (control character) đưa con trỏ về đầu dòng (không bao gồm xuống dòng)
 * `\t`: Tab. Tìm kiếm ký tự tab ngang (tập hợp 1 số spacebar nhất định nào đó)
 * `\0`: Null. Tìm kiếm ký tự null. Một ký tự dùng để kết thúc chuỗi trong văn bản.

#### Anchors

* `\G`: Start of match. Trả về kết quả của lần tìm kiếm trước đó hoặc lần gặp đầu tiên của một chuỗi. Khi bạn sử dụng token này với modifier `g` (global).

![Capture.PNG](/assets/images/posts/1f562f71-bf1e-4451-beb2-e21753759a4c.png)

* `^`: Start of string. Tìm kiếm ở vị trí bắt đầu của một chuỗi. Nếu bạn sử dụng modifier `m` (multipleline) thì nó sẽ tìm kiếm theo vị trí bắt đầu của chuỗi ở mỗi dòng.

![Capture.PNG](/assets/images/posts/c7e998fe-9008-4a4b-85c5-19e9a05eb55e.png)

* `$`: End of string. Tìm kiếm ở vị trí cuối cùng của một chuỗi. Nếu bạn sử dụng modifier `m` thì nó sẽ tìm kiếm theo vị trí cuối cùng của chuỗi ở mỗi dòng.

![Capture.PNG](/assets/images/posts/f8aacae6-c002-4e57-aeee-01b0e8f2b110.png)

* `\A`: Start of string. Cũng là tìm kiếm ở ví trí bắt đầu của mỗi chuỗi. Nhưng không giống với `^`, `\A` không bị ảnh hưởng bởi modifier `m`. Nếu bạn đã làm việc với RoR, khi viết RegEx và chạy Rubocop, nó sẽ khuyên bạn sử dụng `\A` thay vì `^` đấy :D

![Capture.PNG](/assets/images/posts/d0cd421f-8783-4db9-8960-05edf132ab16.png)

* `\Z`: End of string. Cũng là tìm kiếm ở vị trí cuối cùng của chuỗi. Nhưng không giống với `$` vì `\Z` không bị ảnh hưởng bởi modifier `m`. Cũng giống với `\A`, Rubocop sẽ khuyên bạn sử dụng token này thay thế token `$` :D

![Capture.PNG](/assets/images/posts/e06b8a24-9207-4683-b308-42f56871e668.png)

* `\z`: Absolute end of string. Về ý nghĩa thì nó cũng giống với token `\Z`, nhưng nó ngược lại với `\Z`, nó sẽ không tìm kiếm nếu như vẫn còn một ký tự newline đằng sau nó. Bạn có thể xem 2 ảnh của `\Z` và `\z` để so sánh.

![Capture.PNG](/assets/images/posts/86f75bd0-622d-4e79-a12e-23530112fe58.png)

* `\b`: A word boundary. Tìm kiếm chính xác từ đó mà không chứa một ký tự bất kỳ nào khác. Bạn có thể xem ảnh để hiểu rõ hơn. Mình đã viết ví dụ rất rõ ràng để mô tả token `\b` này

![Capture.PNG](/assets/images/posts/e6897853-aa22-4c50-9352-823860265306.png)

* `\B`: Non-word boundary. Ngược lại của token `\b`.

![Capture.PNG](/assets/images/posts/8b26636e-0026-472e-a05b-e9c2a8e72c6e.png)

#### Meta sequences

* `.`: Any single character. Tìm kiếm một ký tự bất kỳ (bao gồm cả ký tự khoảng trắng \s) khác ký tự xuống dòng (newline).
* `\s`: Any whitespace character. Tìm kiếm ký tự khoảng trắng, tab ngang hay một ký tự xuống dòng (newline)
* `\S`: Any non-whitespace character. Ngược lại với token `\s` là tìm kiếm một ký tự không phải là khoảng trắng, tab ngang hoặc ký tự xuống dòng (newline).

![Capture.PNG](/assets/images/posts/5b9ee50d-12bc-4e4d-bbb7-904e170f268f.png)

* `\d`: Any digit. Tìm kiếm số thập phân. Tương đương với `[0-9]`.
* `\D`: Any non-digit. Ngược lại với token `\d`. Là tìm kiếm 1 ký tự không phải là một số thập phân.

![Capture.PNG](/assets/images/posts/920970da-644d-47f8-8b6a-a1c791e70dda.png)

* `\w`: Any word character. Tìm kiếm chữ, số và dấu gạch dưới (underscore) không bao gồm khoảng trắng (whitespace), tab ngang và dấu xuống dòng (newline).
* `\W`: Any non-word character. Ngược lại với token `\s`. Là tìm kiếm một ký tự không phải là chữ, số và dấu gạch dưới (underscore).

![Capture.PNG](/assets/images/posts/56552f15-e329-445a-9972-5c3244a5a2d8.png)

* `\v`: Vertical whitespace character. Tìm kiếm xuống dòng (newline) và ký tự tab dọc. Bạn có thể xem thêm định nghĩa về **vertical tab** [tại đây](https://en.wikipedia.org/wiki/Tab_key)!
* `\n`: Newline character. Tìm kiếm ký tự xuống dòng (newline).
* `\uYYYY`: Hex character _YYYY_. Tìm kiếm ký tự unicode với mã hex.

![Capture.PNG](/assets/images/posts/e87d20b3-6a8a-4765-9821-0d31c1770281.png)

* `\xYY`: Hex character _YY_. Tìm kiếm ký tự 8-bit với mã hex.

![Capture.PNG](/assets/images/posts/3ceb4e67-c317-44be-aee9-b6647d281f66.png)

* `\ddd`: Octal character _ddd_. Tìm kiếm ký tự với mã Oct.

![Capture.PNG](/assets/images/posts/3b1a1a9c-be2d-4cd5-9d0a-6c4375ec66d9.png)

* `cY`: Control character _Y_. Tìm kiếm ký tự ASCII thông thường được gắn kèm với phím Control (Ctrl). Ví dụ: `cC` là tìm kiếm ký tự Control+C
* `[\b]`: Backspace character. Tìm kiếm ký tự điều khiển backspace (trên bàn phím là nút xóa 1 ký tự về bên trái - ngược với phím Del(ete) là xóa 1 ký tự về bên phải)
* `\`: Escape character. Ký tự này được sử dụng để escape 1 ký tự điều khiển nào đó.

![Capture.PNG](/assets/images/posts/662d58d6-40af-403b-88ec-a76aecd0270e.png)

#### Quantifiers

* `?`: Zero or one. Tìm kiếm một ký tự nào đó với điều kiện có hoặc không có cũng được. Ví dụ bạn muốn validate một URL nào đó cho phép cả http hoặc https. Bạn có thể dùng `/^http|https/` cũng được. Nhưng nó khá là thừa. Thay vào đó bạn có thể sử dụng tokens `?` để thực hiện việc này. Trong ảnh dưới đây. Mình có viết 5 ví dụ. Thì ví dụ thứ 3 để mọi người hiểu làm sao sử dụng `?` với số lượng ký tự lớn hơn 1 :D.

![Capture.PNG](/assets/images/posts/d5cf2d75-2d35-43b9-af4c-ba370d58f8ad.png)

* `*`: Zero or more. Tìm kiếm một ký tự nào đó với điều kiện không có hoặc có nhiều hơn một lần.

![Capture.PNG](/assets/images/posts/2a8ecc1b-9023-4ac4-95d1-22bfdcae338e.png)

* `+`: One or more. Tìm kiếm 1 hoặc nhiều ký tự xuất hiện liên tiếp nhau (không giống với `*` là không có hoặc có nhiều).
* `{n}`: Exactly of _n_. Tìm kiếm chính xác số lần (n) xuất hiện của một ký tự nào đó.

![Capture.PNG](/assets/images/posts/b16e6108-a555-4c6c-8cfd-85be735b1ca5.png)

* `{n,}`: _n_ or more. Tìm kiếm chính xác số lần (n) xuất hiện liên tiếp từ _n_ cho đến vô hạn.

![Capture.PNG](/assets/images/posts/614504b2-97ec-4c45-a6ce-c188ac17ac95.png)

* `{n,m}`: Between _n_ and _m_: Tìm kiếm chính xác số lần xuất hiện trong khoảng từ `n` cho đến `m` lần. Mình sẽ lấy ví dụ là 1 validate đơn giản số điện thoại của Việt Nam nhé :D

![Capture.PNG](/assets/images/posts/dd72dc48-a0cc-4422-b022-bf90dfa2a900.png)

* `*`: Greedy quantifier. Tìm kiếm không quan tâm đến việc nó chứa những ký tự gì bên trong mà chỉ cần biết nó có phù hợp với pattern của mình hay không. Bạn nên cẩn thận khi sử dụng pattern kiểu này để tránh việc tìm cả những dữ liệu không mong muốn.

![Capture.PNG](/assets/images/posts/44852f66-43d0-4ac0-933b-e56c18266f6c.png)

* `*?`: Lazy quantifier. Kiểu tìm kiếm một vài ký tự bất kỳ. Không quan tâm đến việc nó chứa gì. Chỉ cần phù hợp với pattern của mình đưa ra.

![Screenshot from 2016-08-20 16:47:50.png](/assets/images/posts/08c051a0-0437-4259-88da-a8a35e838800.png)

#### Group constructs

* `(...)`: Capture everything enclosed. Nhóm (gộp) một hoặc nhiều pattern cần tìm kiếm.

![Screenshot from 2016-08-20 17:29:33.png](/assets/images/posts/78a346dd-db0c-4ca5-8263-f3b9aebd7866.png)

* `(a|b)`: Match either a or b. Tìm kiếm chuỗi nào phù hợp với pattern này hoặc pattern kia. Bạn có thể xem lại ví dụ của `(...)` hoặc ví dụ về validate số điện thoại của phần `{n,m}`.
* `(?=...)`: Positive lookahead. Tìm kiếm pattern trước nó với điều kiện phải phù hợp với subpattern đằng sau nó. Bạn có thể xem ví dụ để hiểu hơn

![Screenshot from 2016-08-20 17:45:48.png](/assets/images/posts/634eb0d2-d7ae-446f-9410-e3e361fc23a3.png)

* `(?!..)`: Native lookahead. Ngược lại với pattern `(?=...)`. Tìm kiếm pattern trước nó với điều kiện không được giống (phủ định) subpattern đằng sau nó.

![Screenshot from 2016-08-20 17:51:58.png](/assets/images/posts/16a0ebfb-c61b-4b4e-b1b5-dc634c93ed3b.png)

* `(?<=...)`: Possitive lookbehind. Tìm kiếm một giá trị mà có giá trị trước đó phù hợp với pattern được đưa ra trong cặp `(?<=...)`.

![Screenshot from 2016-08-21 16:32:58.png](/assets/images/posts/14aa22be-e4f8-4b83-80ef-280772b6a198.png)

* `(?<!...)`: Negative lookbehind. Ngược lại với `Positive lookbehind` là tìm kiếm một giá trị mà có giá trị trước đó không giống (phủ định) với giá trị đưa ra trong cặp `(?<!...)`.

![Screenshot from 2016-08-21 16:44:55.png](/assets/images/posts/0e875a81-b13b-4346-9663-d48a96e0c35e.png)

#### Character classes

* `[abc]`: Single character. Tìm kiếm một trong các ký tự được xuất hiện trong pattern.

![Screenshot from 2016-08-21 16:48:13.png](/assets/images/posts/456f5961-7ab8-4793-a75c-9cb7726fceb7.png)

* `[^abc]`: Single character but except pattern. Tìm kiếm các ký tự không thuộc một trong các ký tự được xuất hiện trong pattern.

![Screenshot from 2016-08-21 16:50:39.png](/assets/images/posts/87ad0182-e407-4573-a762-bfddae480574.png)

* `[a-z]`: A character in the range. Tìm kiếm một hoặc các ký tự nằm trong khoảng pattern.

![Screenshot from 2016-08-21 16:53:01.png](/assets/images/posts/5a1ffbf9-d96b-4633-8066-dcebe8373d04.png)

* `[^a-z]`: A character not in the range. Tìm kiếm một hoặc các ký tự không nằm trong khoảng pattern.

![Screenshot from 2016-08-21 16:54:52.png](/assets/images/posts/1c614bfa-bff1-4ae6-8698-1b86fe818438.png)

* `[[:alnum:]]`: Letters and digits. Tìm kiếm tất cả các chữ và số. Tương đương với `[a-zA-Z0-9]`
* `[[:alpha:]]`: Lettes. Tìm kiếm tất cả các chữ cái alpha. Tương đương với `[a-zA-Z]`
* `[[:ascii:]]`: ASCII codes from 0 to 127. Tìm kiếm các ký tự nằm có mã ASCII nằm trong khoảng từ 0-127. Bạn có thể tham khảo ASCII codes table [**tại đây**](http://www.ascii-code.com/).
* `[[:blank:]]`: Space or tab only. Chỉ tìm kiếm dấu khoảng trắng hoặc dấu tab ngang.
* `[[:cntrl:]]`: Control character. Tìm kiếm các ký tự điều khiển, bao gồm dấu ngắt dòng (newline), ký tự null (null character), tab và ký tự thoát (escape character).
* `[[:digit:]]`: Decimal digits. Tìm kiếm chữ số thập phân. Tương tự với pattern `[0-9]`.
* `[[:lower:]]`: Lowercase letters. Tìm kiếm các ký tự chữ thường. Tương tự với `[a-z]`.
* `[[:punct:]]`: Visible punctuation character. Tìm kiếm các ký tự đặc biệt trong khoảng ASCII code từ 0-127 không bao gồm khoảng trắng.
* `[[:space:]]`: Whitespace. Tìm kiếm dấu khoảng trắng (spacebar). Tương tự `\s`.
* `[[:upper:]]`: Uppercase letters. Tìm kiếm các ký chữ hoa. Tương tự `[A-Z]`
* `[[:word:]]`: Word character. Tìm kiếm các chữ, số và dấu gạch dưới. Tương tự `\w`.
* `[[:xdigit:]]`: Tìm kiếm số chữ số thập lục phân. Tương tự `[0-9A-Fa-f]`. Dùng để tìm kiếm mã màu trong CSS rất tiện ^^!

![Screenshot from 2016-08-21 17:28:50.png](/assets/images/posts/5078699b-5ff1-4d4a-99ca-eb53bcd2ecdd.png)

### RegEx modifier

**RegEx modifier** là gì? Nó là các cờ (flag) điều khiển, thông báo rằng bạn muốn thực hiện pattern của mình như thế nào đối với dữ liệu đầu vào. Ví dụ, bạn muốn tìm kiếm tất cả các ký tự từ a-z bao gồm cả hoa và thường, thì bạn có thể sử dụng pattern `[a-zA-Z]`. Nhưng bạn có thể rút ngắn pattern của mình bằng cách sử dụng modifier `i`(nsensitive) như sau: `/[a-z]+/i`.

* `g`: **g**lobal. Tìm kiếm từ đầu cho đến cuối. Gặp kết quả thì trả ra và tiếp tục chứ không dừng lại.
* `m`: **m**ultiple-line. Tìm kiếm trên nhiều dòng. Sử dụng `^` và `$` để tìm kiếm pattern đó trên mỗi dòng chứ không phải là tìm kiếm từ vị trí bắt đầu đến kết thúc của một chuỗi.
* `i`: **i**nsensitive. Bỏ qua phân biệt hoa thường.
* `x`: e**x**tended. Sử dụng để viết comment trong pattern.

![Screenshot from 2016-08-21 17:53:59.png](/assets/images/posts/5ff62303-a214-47ed-bf19-f3eb2ba8fb66.png)

* `X`: e**X**tra. Khi bạn sử dụng modifier này thì ký tự sau dấu `\` (escape) mà là một ký tự đặc biệt thì sẽ không được escape nữa mà sẽ báo lỗi.

![Screenshot from 2016-08-21 18:00:11.png](/assets/images/posts/59d36c4a-da3c-421e-9b55-35855ffc82bb.png)

* `s`: **s**ingle-line. Hay còn gọi là DOTALL. Sử dụng modifier này khi bạn muốn tìm kiếm tất cả, bao gồm cả dấu xuống dòng (newline). Bạn có thể xem ảnh ví dụ để hiểu rõ hơn

![Screenshot from 2016-08-21 20:05:33.png](/assets/images/posts/5ea2c0e2-d3a8-40ec-a6ca-9b177d87d472.png)

![Screenshot from 2016-08-21 20:06:05.png](/assets/images/posts/25824ea1-cccc-4439-9624-9cfb0928b593.png)

* `U`: **U**ngreedy. Modifier này đảo ngược lại quá trình tìm kiếm của quantifier `+`, hay còn gọi là tắt bỏ tính năng mặc định của modifier này. Khi bạn muốn bật nó lên trong lúc sử dụng modifier này thì bạn chỉ cần thêm `?` là được. Modifier này khá là khó giải thích. Nên mình sẽ sử dụng hình ảnh để minh họa nhé :D!

![Screenshot from 2016-08-24 20:21:25.png](/assets/images/posts/8c3daae4-bb2f-4e04-8596-f8f30096360c.png)

![Screenshot from 2016-08-24 20:21:50.png](/assets/images/posts/ce7d5e0e-1438-4f6f-b1ed-fb21755ff36b.png)

![Screenshot from 2016-08-24 20:22:09.png](/assets/images/posts/5dc00c86-5a39-4142-b306-863c032381e1.png)

![Screenshot from 2016-08-24 20:22:31.png](/assets/images/posts/19d7e72a-d106-4f40-9cdf-3314ccf9a7fe.png)

### Lời kết

Trong bài viết ngăn này, mình giới thiệu đơn giản về các tokens và các modifier được RegEx hỗ trợ. Nhưng không phải ngôn ngữ nào cũng sử dụng giống nhau, vì vậy bạn cũng nên tìm hiểu xem ngôn ngữ bạn đang sử dụng có hỗ trợ modifier hay tokens đó hay không trước khi sử dụng nhé. Ví dụ như modifier `g`(lobal) được hỗ trợ bởi JavaScript nhưng trong PHP thì lại không :D!

Bài viết của mình đến đây xin được phép kết thúc. Hy vọng bài viết này sẽ có ích với bạn :D!
