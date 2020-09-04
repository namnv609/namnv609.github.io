---
date: 2015-12-20 14:53:00 +0700
title: Software cracking
---

### Software cracking là gì?

Khái niệm **Software cracking** chỉ đơn giản là việc sửa hay can thiệp vào phần mềm nào đó nhằm mục đích có được license (giấy phép) hay loại bỏ module kiểm tra license của nó để có thể sử dụng đầy đủ các chức năng một cách không chính thức.<!--more--> Do đó, các kiến thức lập trình tối thiểu là điều yêu cầu phải có. Khi thực hiện việc crack phần mềm, chúng ta sẽ sử dụng một kỹ thuật được gọi là reverse engineering (dịch ngược phần mềm sang một loại ngôn ngữ nào đó - như assembly chẳng hạn) và khai thác lỗ hổng của nó để viết crack, path, keygen, ...!

Bây giờ chúng ta sẽ đi tìm hiểu một số khái niệm cơ bản của vấn đề này nhé :D!

### Software protection and licensing

Một software sau khi được đưa ra thị trường với mục đích thương mại thì luôn được xử lý để nhằm tránh việc ăn cắp bản quyền gây thiệt hại cho nhà sản xuất. Vậy, việc xử lý này được tiến hành như thế nào? Nó là một module nhỏ nhằm xác định tính chủ quyền của những ai đã mua sản phẩm này. Có thể gọi nó là module protect license.

Module này có rất nhiều hình thức biến hoá nhằm ngăn chặn đến mức tối đa việc xâm phạm bản quyền (bao gồm việc sử dụng phần mềm ở mức độ không được sự cho phép của nhà sản xuất, hoặc ăn cắp các giải thuật để code lại phần mềm khác nhằm cạnh trạnh, ...)! Mình sẽ giới thiệu một số kiểu bảo vệ bản quyền phần mềm thông dụng nhé.

* **Serial**: Là một chuỗi ký tự mà phần mềm yêu cầu bạn nhập vào để đăng ký phần mềm đó để có thể sử dụng đầy đủ các tính năng và không bị giới hạn thời gian sử dụng. Và các chuỗi serial này phụ thuộc vào tiêu chuẩn và sắp đặt riêng của các nhà sản xuất. Và trong chính bản thân loại này cũng được phân loại theo nhiều hình thức khác nhau như:
    * Số serial là một giá trị cố định được lưu ở một hằng số nào đó trong phần mềm. Khi bạn đăng ký (mua) thì sẽ được cung cấp chuỗi này để đăng ký phần mềm.
    * Số serial phụ thuộc vào thông tin đăng ký như tên, email, ... Nó sẽ phụ thuộc vào các giá trị này để tính toán ra một số serial phù hợp với nó.
    * Số serial phụ thuộc vào thông tin máy tính cài nó như các thông số máy, thông số phần cứng, ... Và với dạng này, mỗi license chỉ sử dụng được duy nhất trên một máy tính. Điển hình là Windows, khi bạn thay đổi phần cứng thì serial cũ mà bạn đã sử dụng trước đó sẽ không sử dụng được nữa.
    * Số serial được kiểm tra online. Về bản chất, nó cũng giống các loại serial khác, nhưng sử dụng hình thức kết nối tới server của nhà sản xuất, nhận thông tin trả về để đăng ký phần mềm. Phổ biến và quen thuộc nhất của kiểu này là phần mềm IDM (Internet Download Manager) và Windows.

Còn nhiều loại bảo vệ nữa, nhưng do giới hạn của kiến thức bản thân, nên mình chỉ xin phép giới thiệu mấy loại trên thôi nhé :)! Giờ chúng ta sẽ đi tìm hiểu các kiểu crack quen thuộc nhé.

* Keygen (Key generator): Là một chương trình nhỏ tạo ra số serial để đăng cho phần mềm. Được viết bởi các cracker sau khi đã tìm ra cách tính toán số serial của phần mềm đó.
* Path: Là một chương trình tác động vào phần mềm nguồn (sửa chữa cấu trúc, mã nguồn) nhằm đăng ký mà không cần phải đi qua bước nhập số serial
* REG File: Là một file với phần mở rộng là `.reg` (registry). Khi chạy file này, nó sẽ thêm vào Registry của Windows các thông tin cần thiết để đăng ký phần mềm.
* Loader: Là một dạng chương trình chạy trước chương trình cần đăng ký mỗi khi sử dụng nó. Nó sẽ biến chương trình thành đã được đăng kí, và mỗi lần sử dụng bắt buộc lại phải chạy loader trước.
* Key Maker: Là chương trình tạo ra file đăng ký của riêng từng chương trình (dạng .key chẳng hạn). Và điển hình của loại này là thằng WinRAR :D!
* Cracked .exe file: Là file thực thi của phần mềm đã bị bẻ khóa sẵn, ta phải ghi đè nó lên file .exe gốc của phần mềm để sử dụng.

Bây giờ mình sẽ giới thiệu một số công cụ thường được sử dụng để thực hiện công việc này nhé.

* W32Dasm: Giúp chúng ta dịch ngược file exe, dll, com, ... sang mã assembly. Mình dùng thằng này với mục đích đảo ngược đoạn mã kiểm tra serial (từ nếu đúng sẽ thành đăng ký bản quyền sang sai cũng được chấp nhận (hehe)).
* OllyDbg: Cũng giống với W32Dasm, nhưng nó có nhiều thông tin chi tiết hơn ngoài mã assembly như CPU, FPU, Stack, Memory, ... Mình dùng nó với mục đích tìm số serial chứ không phải đảo ngược quá trình kiểm tra.
* PEiD: Giúp bạn detect loại packer mà phần mềm sử dụng. Ngoài ra, nếu file không bị pack thì nó sẽ cho bạn biết rằng phần mềm này sử dụng ngôn ngữ gì!
* HIEW: Giúp bạn xem và sửa mã hex của chương trình cần bẻ khoá.

Tiếp theo, mình sẽ giới thiệu một số lệnh và chỉ thị căn bản của Assembly để tiện cho việc các bạn có thể hiểu trong phần thực hành nho nhỏ sắp tới nhoé :D!

### Assembly

Các chỉ thị căn bản:

* MOV dest, src: Chuyển giá trị từ toán hạng nguồn vào toán hạng đích
* PUSH src: Cất giá trị toán hạng nguồn vào đỉnh ngăn xếp
* POP dest: Chuyển giá trị từ đỉnh ngăn xếp vào toán hạng đích
* LEA reg, mem: Đưa địa chỉ Offset của toán hạng nguồn vào thanh ghi đích
* ADD dest, src: Cộng toán hạng nguồn và đích. Kết quả lưu ở toán hạng đích
* SUB dest, src: Trừ toán hạng nguồn và đích. Kết quả lưu ở toán hạng đích
* MUL dest, src: Nhân toán hạng nguồn và đích. Kết quả lưu ở toán hạng đích
* DIV dest, src: Chia toán hạng nguồn và đích. Kết quả lưu ở toán hạng đích
* INC dest: Tăng 1 cho toán hạng đích
* NOT dest: Đảo bit
* AND dest, src: Phép và logic giữa toán hạng đích và nguồn. Kết quả trả về ở toán hạng đích
* OR dest, src: Phép hoặc logic
* TEST dest, src: So sánh nội dung 2 toán hạng qua phép toán AND. Kết quả tác động đến các cờ

Một số câu lệnh nhảy có điều kiện cơ bản:

* JA: Jump if above
* JAE: Jump if above or equal
* JB: Jump if below
* JBE: Jump if below or equal
* JE: Jump if equal
* JNE: Jump if not equal
* JZ: Jump if zero
* JNZ: Jump if not zero
* JMP: Jump Unconditionally

Sang phần tiếp theo, **Think to crack** nhé :D!

### Think to crack

Như đã giới thiệu ở trên, phần mềm thương mại khi được phát hành sẽ có những module protect license khác nhau. Và vấn đề ở đây là làm sao chúng ta có thể qua được những module này để có thể sử dụng chương trình một cách đầy đủ nhất. Để được như vậy, chúng ta cần phải hiểu module protect đó là loại gì, nó diễn ra như thế nào trong chương trình khi nó được thực thi. Để có thể hiểu được như vậy, việc chúng ta cần làm là phải đọc nó để rút ra được cách chống lại chính nó.

Việc đọc quá trình này sẽ dễ dàng nếu như chúng ta có source code của phần mềm đó trong tay. Nhưng thực tế thì không khả thi vì không ai cung cấp cho chúng ta thứ đấy cả :))! Vì vậy, chúng ta sẽ phải đọc nó ở một dạng ngôn ngữ khác, đó là Assembly - một dạng ngôn ngữ máy. Sau khi dịch ngược phần mềm sang mã máy, chúng ta sẽ trace đến module protect này. Và phần này rất đa dạng nên chúng ta cần phải linh động thay đổi hướng hướng giải quyết nhằm phù hợp với các trường hợp cụ thể.

Rút gọn lại, chúng ta sẽ làm những bước chính như sau: Tìm cách set breakpoint (mình sẽ giới thiệu nó sau) từ các API thông dụng của Windows như: GetWindowTextA, GetDlgItemTextA, MessageBoxA. Vì sao vậy? Vì trong đa phần trường hợp nhập thông tin đăng ký phần mềm vào thì chương trình luôn lấy những thông tin ta nhập (GetWindowTextA, GetDlgItemTextA), sau đó kiểm tra và trả về thông báo (MessageBoxA, MeesageBeep) của các thông tin đó là đúng hay sai. Vì vậy, với cách này, chúng ta sẽ đến được ngay trước quá trình xử lý thông tin mà chúng ta đã nhập (tức là trước khi module protect license được gọi). Hoặc đơn giản hơn là chúng ta sử dụng câu thông báo trả về (như: "Invalid serial number") để lần ngược về quá trình xử lý của nó. Hoặc một số API khác như: RegQueryValue, RegOpenKeyExA, ReadFile, ... để xử lý với trường hợp dùng RegKey hay KeyFile.

* Breakpoint: hay còn được gọi là "điểm ngắt" - tức là ngắt (hay dừng) một tiến trình đang hoạt động tại một vị trí nào đó, từ đó có thể kết xuất giá trị của một vài hoặc tất cả các biến của chương trình. Điểm ngắt còn có thể được thiết lập bởi các lập trình viên như là một sự tương tác với công cụ gỡ rối. Nói chung điểm ngắt được sử dụng để dừng quá trình thực thi của một chương trình.

Okie, lý thuyết suông thế đủ rồi. Bây giờ chúng ta sẽ thực hành xíu nhé :D! Đối tượng đem ra thực hành sẽ là phần mềm [**Sweet MIDI Player 2.6.7**](http://www.ronimusic.com/), có giá là 14.95$. Với phần mềm này, nó sử dụng cơ chế sinh mã serial theo User ID cho từng máy. Chúng ta sẽ đi tìm số serial phù hợp để đăng ký nhé. Mình sẽ dùng OllyDbg để thực hiện việc này. Đầu tiên, chúng ta tải về, cài đặt và chạy như bình thường. Bụp, một hộp thoại bắt ta nhập Password dựa theo User ID mà nó đưa cho. Tất nhiên, chúng ta cứ nhập bừa vào (tỷ lệ đúng là 1/∞) để xem message nó bắn ra như thế nào.

![Capture.PNG](/assets/images/posts/7e886afd-cd10-4950-9441-c9ec760a19e2.png)

OK, nhớ lấy message đó (Not a valid password)! Chúng ta tắt chương trình này đi. Mở nó lên bằng OllyDbg. Tiếp theo, nhấn F9 để chạy chương trình, rồi nháy phải chuột trong khung CPU rồi chọn **Search for** -> **All referenced text string**

![Untitled.png](/assets/images/posts/cba5f5ac-2f09-478a-b88f-f5493b50bedb.png)

Trong cửa sổ **Text string referenced in Swmipl32**, chúng ta tìm đến đoạn string "Not a valid password!". Khi tìm thấy, chúng ta chọn nó rồi nhấn Enter. Khi đó, chúng ta sẽ ở địa chỉ chứa đoạn text trên (0132E289 - hoặc có thể khác khi ở máy của mọi người). Thông thường, chúng ta chỉ cần lần ngược lên trên, tìm đến đoạn nào có lệnh JE (hoặc JNE) rồi đảo ngược chúng nó lại (JE -> JNE và JNE -> JE), chạy phần mềm và đăng ký với thông tin lung tung, có thể sẽ thành công. Về cơ bản, thằng này cũng thế, bạn chỉ cần đi lên trên một chút sẽ thấy đoạn:

```
0132E23C     74 4B          JE SHORT Swmipl32.0132E289
```

rồi double click vào, sửa JE thành JNE và bấm F9 một lần nữa để chạy lại và đăng ký là xong, nhưng nó yêu cầu bạn phải khởi động lại phần mềm, và nó sẽ kiểm tra số serial đó khi nó được khởi động lại -> vẫn không hợp lệ. Vì vậy, chúng ta cần phải đi tìm số serial thật sự của nó. OK, quay lại địa chỉ 0132E289. Chúng ta sẽ đặt breakpoint tại đây, trong OllyDbg, nhấn F2 để đặt (tắt - toggle) breakpoint. Sau khi đặt xong, chúng ta nhấn F9 để chạy chương trình. Nhập password vào và bấm đăng ký, khi này sẽ không xuất hiện thông báo invalid password nữa, mà chúng ta sẽ dừng lại ở địa chỉ 0132E289 do chúng ta đã đặt breakpoint ở đó. Tiếp theo, chúng ta nhấn F8 để step over vào hàm xử lý đoạn này, nhìn xuống góc bên phải của OllyDbg, chúng ta sẽ thấy:

![Capture_01.PNG](/assets/images/posts/be406be2-4fea-4f09-ab62-e4c067ef8f07.png)

Okie, cuộn chuột lên trên một chút, chúng ta thấy dãy số mà mình nhập vào (123456) xuất hiện ở đây. Có cơ may rồi, cuộn tiếp, biết đâu chúng ta sẽ tới được đoạn mà nó sẽ so sánh chuỗi đúng với chuỗi của chúng ta (hehe)! Vâng, đúng thật, chỉ lên thêm một chút và chúng ta đã gặp chuỗi của mình và chuỗi của họ. Không chỉ một, mà rất nhiều (cuoideu)!

![Capture_002.PNG](/assets/images/posts/611fe3f0-3244-4748-9cc4-85926898419b.png)

Lấy thử chuỗi này (BA053FE6) để đăng ký xem sao nhé. Tiếp tục nhấn F9 để chạy tiếp, chúng ta nhập lại cái chuỗi vừa tìm được xem sao. Olala, được rồi này (haha)!

![Capture_003.PNG](/assets/images/posts/c836718e-0480-401c-986f-6bc4b8009428.png)

Giờ thử khởi động lại phần mềm xem nó có hỏi nữa không nhé (cuoideu)!

### Lời kết

Vì khái niệm crack này không thể giải thích và giới thiệu hết trong một bài viết được. Nên mình chỉ xin phép giới thiệu những khái niệm căn bản của nó. Và cũng một phần là do kiến thức của mình vẫn còn hạn hẹp nên không thể truyền tải được hết trong bài viết này. Mong mọi người thông cảm.

**Bài viết này chia sẻ không nhằm với mục đích tuyên truyền việc sử dụng phần mềm không có bản quyền. Bài viết chỉ chia sẻ với mục đích tìm hiểu về kỹ thuật reverse engineering và bảo mật phần mềm. Xin cảm ơn!**
