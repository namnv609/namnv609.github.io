---
date: 2015-06-15 10:24:00 +0700
title: Introduction to Gith hooks
---

### Githook là gì?

Giống như các hệ thống quản lý version khác, Git cũng cung cấp cho chúng ta một cách để can thiệp vào một số quá trình đặc biệt của nó bằng những custom script, đó là hook.<!--more-->

Git hook có 02 nhóm là:
* Hook cho client
  * Là những hook dành cho những quá trình được thực hiện ở phía client như commit hay merge
* Hook cho server:
  * Là những hook dành cho những quá trình được thực hiện ở phía server như quá trình nhận được một commit push lên từ client, ...

Trong bài viết này mình sẽ giới thiệu về 02 nhóm hook này, cách cài đặt, sử dụng và cách viết hook cho riêng mình.

### Cài đặt hook

Các hook của Git được lưu trong thư mục `git_project/.git/hooks`. Mặc định, Git cung cấp một loạt các ví dụ cho từng hook trong thư mục hooks. Mọi người có thể mở nó bằng **VIM** hay một editor bất kỳ để xem chi tiết.

Các ví dụ được viết bằng shell script, tuy nhiên bạn có thể viết nó bằng bất cứ ngôn ngữ nào (ví dụ như **Python**, **Ruby**, ...), miễn là bạn có thể chạy nó như một file thực thi (executable) bằng cách khai báo môi trường thực thi ở dòng đầu tiên của hook file. Ví dụ:

```sh
#!/usr/bin/env bash
```

hay

```sh
#!/usr/bin/env python
```

hoặc

```sh
#!/usr/bin/env php
```

Tất cả các tên file ví dụ là dành cho các hook tương ứng trong Git và có kết thúc là `.example`. Để áp dụng nó, bạn chỉ cần đổi tên file bằng cách bỏ `.example` và set cho hook đó có quyền thực thi là được.
Ví dụ, bạn muốn sử dụng hook `pre-commit` thì bạn chỉ cần làm như sau:

``mv .git/hooks/pre-commit.sample .git/hooks/pre-commit; chmod +x .git/hooks/pre-commit``

Bây giờ chúng ta sẽ đi tìm hiểu từng hook của mỗi nhóm hook nhé!

### Client-side hooks

Có rất nhiều loại client-side hooks, chúng ta sẽ chia nó theo workflow.

#### Committing-Workflow hooks

* `pre-commit`: Hook này được gọi đầu tiên, thực thi trước khi bạn nhập nội dung cho commit message. Hook này được sử dụng để kiểm tra nội dung các tập tin được commit. Bạn có thể viết script để kiểm tra code convention, run test hoặc chạy static analysis trước khi commit. Nếu script trả về kết quả lớn hơn 0. Quá trình commit sẽ bị hủy bỏ. Tuy nhiên bạn có thể bỏ qua hook này bằng cách truyền tham số --no-verify vào lệnh commit.
* `prepare-commit-msg`: Là hook được chạy trước khi trình soạn thảo commit message (ví dụ như Vim hay Gedit) được gọi, nhưng lại sau khi message mặc định được tạo ra. Hook này giúp bạn thay đổi message mặc định trước khi chúng ta thấy nó. Hook này cơ bản ít khi sử dụng đến, chỉ trừ trường hợp các commit message được sinh ra tự động như quá trình **squashed** hay **merged** các commit.
* `commit-msg`: Hook này về cơ bản là gần giống với hook `prepare-commit-msg`, nhưng nó được gọi sau khi chúng ta đã nhập nội dung message cho commit. Thích hợp trong việc chuẩn hóa commit message. Chỉ có một tham số được nhận trong hook này, đó là tên file chứa commit message. Nếu script này trả về kết quả lớn hơn 0 thì quá trình commit sẽ bị hủy.
* `post-commit`: Được chạy khi quá trình commit hoàn tất. Hook này không thể thay đổi kết quả của quá trình commit. Vì vậy, nó chỉ dùng cho mục đích thông báo đến người dùng. Hook này không nhận tham số và kết quả trả về từ script cũng không tác động đến quá trình commit như ba hook trên.

#### Email workflow hooks

Bạn có thể setup 03 client site hooks cho email workflow. Tất cả các hook này đều liên quan đến lệnh `git am`. Nếu bạn không sử dụng câu lệnh này trong workflow thì bạn có thể bỏ qua phần này. Nhưng nếu bạn nhận patch qua email được chuẩn bị bởi lệnh `git format-patch` thì có thể những hook này sẽ có ích với bạn.
* `applypatch-msg`: Hook này nhận một tham số là tên file tạm chứa nội dung của commit message. Git sẽ bỏ qua quá trình patch nếu hook này trả về kết quả lớn hơn 0. Bạn có thể sử dụng hook này với mục đích đảm bảo commit message là đúng chuẩn.
* `pre-applypatch`: Hook này không nhận tham số nào và tương đối giống với `pre-commit`. Nó được thực thi sau khi một patch được áp dụng. Vì thế bạn có thể sử dụng nó để kiểm tra toàn bộ source code trước khi tạo commit.
* `post-applypatch`: Bạn có thể sử dụng hook này để gửi thông báo tới mọi người trong nhóm hay gửi thông báo cho tác giả về bản patch này.

#### Một vài client-side hooks khác

* `pre-rebase`: Hook này được thực thi trước khi câu lệnh `git rebase` thực hiện việc thay đổi. Sử dụng để đảm bảo không có lỗi gì xảy ra. Hook này nhận 02 tham số là upstream branch và branch sẽ được rebase. Nếu bạn rebase với branch hiện tại thì có thể bỏ trống tham số thứ 2. Quá trình rebase sẽ bị hủy nếu script trả về kết quả lớn hơn 0.
* `post-checkout`: Hook này có nhiều điểm giống với hook **post-commit**, nhưng nó được chạy sau khi checkout thành công. Bạn có thể sử dụng nó để setup môi trường hay sinh document sau khi checkout. Hook nhận 03 tham số là ref của HEAD trước, ref của HEAD mới và một cờ (1 và 0) để thông báo đó là branch hay file. Kết quả trả về từ script sẽ không tác động đến quá trình checkout.
* `post-merge`: Hook này được thực thi sau khi bạn chạy lệnh `git merge` thành công. Bạn có thể sử dụng hook này để khôi phục lại các dữ liệu trong thư mục làm việc mà Git không thể kiểm tra như dữ liệu liên quan đến permission.

### Server-side hooks

Bên cạnh việc sử dụng các client-side hooks, bạn có thể sử dụng một vài server-side hooks để áp đặt một cơ chế quản lý cho project của mình. Những hook này được thực thi trước và sau khi commit được đẩy lên server. Những pre-hook có thể hủy bỏ các quá trình liên quan hay in ra một message lỗi cho client nếu như script trả về một kết quả lớn hơn 0.
* `pre-receive`: Hook này được chạy khi server nhận được một push từ phía client. Hook này nhận một danh sách các ref sẽ được push từ stdin và nó sẽ không nhận bất kỳ ref nào nếu như script trả về kết quả lớn hơn 0. Bạn có thể sử dụng hook này để làm những việc như đảm bảo không có bất kỳ ref nào là `non-fast-forwards`.
* `post-receive`: Hook được chạy sau toàn bộ quá trình được hoàn tất. Nó cũng nhận dữ liệu như `pre-receive` từ stdin. Và hook này có thể sử dụng để cập nhật các dịch vụ, để thông báo tới các user như việc gửi email tới các developer liên quan hay là để update tickit tracking system. Thậm chí bạn có thể parse một commit message để kiểm tra xem có bất kỳ tickit nào cần open, modified hay update không. Kết quả trả về từ hook này không tác động đến quá trình push, nhưng phía client sẽ không đóng kết nối tới server cho đến khi quá trình này kết thúc. Vì vậy, bạn nên cẩn thận khi sử dụng hook này với các task tốn thời gian hoặc nếu muốn, thì cách tốt nhất là sử dụng một background process để xử lý các quá trình tốn thời gian này.
* `update`: Hook này tương tự với `pre-receive` ngoại trừ việc nó chỉ chạy các luồng riêng biệt cho mỗi branch mà ta muốn update. Nếu ta muốn update nhiều branch, `pre-receive` chỉ chạy duy nhất một lần, còn `update` sẽ chạy một lần cho mỗi branch mà chúng được push. Hook này không đọc dữ liệu từ stdin như 02 hook trên, mà nó nhận vào 03 tham số gồm: tên branch, SHA-1 trỏ đến commit trước khi được push và SHA-1 mà chúng ta muốn push. Nếu script trả về kết quả lớn hơn 0 thì chỉ branch ứng với script bị loại bỏ (reject) còn các branch khác vẫn được update bình thường.

Trên đây mình đã giới thiệu về git hook và một số hook thường dùng. Nếu bạn muốn tìm hiểu thêm các hook khác, bạn có thể vào trang [Git SCM](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) để xem thêm. Hy vọng bài viết này sẽ giúp bạn tìm thấy một vài thông tin hữu ích giúp bạn tự động hóa một số công việc hàng ngày khi làm việc với hệ thống Git.

Còn bây giờ, chúng ta sẽ thử làm một ví dụ thực tế nhé. Git flow của công ty chúng ta thường là **01 commit/01 branch/01 pull request**. Nhiều khi chúng ta phải commit tạm các thay đổi rồi checkout sang branch khác để làm việc. Đến khi quay lại branch có commit tạm kia, chúng ta quên mất rằng mình vừa commit. Đến khi push code, chúng ta vẫn vô tư commit tiếp rồi push. Nó sẽ có nhiều hơn 01 commit cho branch đấy. Ta sẽ viết một hook trước khi commit, nó sẽ kiểm tra xem branch này đã có commit chưa, nếu có rồi, sẽ thông báo và hỏi chúng ta xem có muốn tiếp tục không. Hook `pre-commit` sẽ phù hợp với yêu cầu này. Okie, let's go.

Đầu tiên, chúng ta sẽ tạo một file với tên `pre-commit` trong thư mục hooks và khai báo môi trường thực thi cho script này. Mình sẽ sử dụng ngôn ngữ shell script để viết nó:

```sh
#!/usr/bin/env bash
```

Tiếp theo, chúng ta sẽ khởi tạo biến chứa tên branch chính (thông thường công ty chúng ta sẽ là branch **develop**, nếu khác thì bạn hãy thay đổi cho phù hợp nhé):

```sh
mainBranchName="develop"
```

Okie, giờ chúng ta lấy tên branch hiện tại:

```sh
currentBranchName=$(git rev-parse --abbrev-ref HEAD)
```

Đếm số lượng commit trong branch hiện tại, tính từ branch chính mà chúng ta checkout ra:

```sh
commitNumber=$(git rev-list --count "$mainBranchName".."$currentBranchName")
```

OK, sau khi đã có số lượng commit. Chúng ta cần phải kiểm tra nó. Nếu số lượng commit của branch hiện tại mà lớn hơn 0 (tức là đã commit rồi, mà chúng ta quên, lại commit tiếp), thì hiển thị message hỏi xem có muốn tiếp tục không:

```sh
if [[ "$commitNumber" > "0" ]]; then
  # Dừng màn hình, đợi người dùng nhập
    # Nếu không có dòng này, quá trình commit vẫn diễn ra
    # mà người dùng không kịp chọn
    exec < /dev/tty

    # Đọc dữ liệu được nhập từ người dùng
    read -p "This branch has more than one commit. Do you want to continue (Y/N)?: " reply

    # Kiểm tra xem ký tự người dùng đã nhập là gì.
    # Nếu khác Y hoặc y thì sẽ hủy bỏ quá trình commit
    # bằng cách trả về một số lớn hơn 0.
    if ! [[ "$reply" =~ ^[Y|y]$ ]]; then
      # Báo cho họ câu cho đỡ shock :))
        echo "Commit aborted!!!!!!!!!!!!!!"
        exit 1
    fi
fi
```

Vậy là xong. Mọi người có thể thử xem sao :D! Nếu có lỗi gì trong ví dụ trên thì mọi người vui lòng để lại comment để cùng tìm hiểu và giải quyết nhé ^^!

Source code: [https://github.com/namnv609/git-hooks/blob/master/pre-commit/one-commit-per-branch](https://github.com/namnv609/git-hooks/blob/master/pre-commit/one-commit-per-branch)
