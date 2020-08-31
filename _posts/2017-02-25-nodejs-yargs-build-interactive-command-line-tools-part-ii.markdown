---
date: 2017-02-25 20:39:00 +0700
title: Node.JS Yargs - Build interactive command line tools (Part II)
---

Như trong [**phần I**](/posts/nodejs-yargs-build-interactive-command-line-tools.html) mình đã giới thiệu qua về module Yargs của Node.JS cũng như giới thiếu một số methods của nó. Hôm nay mình tiếp tục chia sẻ tiếp những methods còn lại của module Yargs này nhé.<!--more-->

## Yargs methods

### `.command(cmd, desc, [Builder], [Handler])`, `.command(cmd, desc, [Module]`, `.command(Module)`

Một ứng dụng to gồm rất nhiều chức năng (module) để thực hiện các công việc khác nhau. Method `.command` giúp chúng ta chia nhỏ ứng dụng thành nhiều module nhỏ thực hiện các chức năng khác nhau. Ví dụ như [**NPM**](https://www.npmjs.com/) có các câu lệnh như `install` để cài một package, `uninstall` để gỡ bỏ một package hay `ls` để xem danh sách các package đã cài (trong thư mục `node_modules`), ...!

Method này có thể nhận các tham số sau (không bắt buộc):

* `cmd`: String hoặc mảng các tên lệnh và alias của các lệnh đó
* `desc`: Mô tả cho lệnh (hoặc các lệnh). Bạn có thể set giá trị của nó thành `false` để tạo ra 1 câu lệnh ẩn (không xuất hiện trong phần trợ giúp của ứng dụng).
* `builder`: Object chứa các gợi ý về các option mà câu lệnh đó chấp nhận.

```javascript
// command-builder.js

var argv = require("yargs")
  .command('install', 'Install package', {
    alias: 'i'
  })
  .help()
  .argv;

/**
$ node command-builder.js --help
Commands:
install  Install package

Options:
--help  Show help                                                    [boolean]
*/
```

**Lưu ý**: Các command sẽ không kế thừa các tùy chọn hay các cài đặt của parent context. Vì vậy, với mỗi command bạn cần phải cài đặt và khai báo lại các tùy chọn hay các cấu hình.

`builder` cũng có thể là 1 function. Function này sẽ thực thi với instance của `yargs` và nó có thể được sử dụng để cung cấp command nâng cao cụ thể nào đó.

```javascript
// command-builder-fn.js
var argv = require("yargs")
  .command("install", "Install package", function(yargs) {
    return yargs.option("name", {
      alias: "n",
      describe: "Package name"
    });
  })
  .help()
  .argv;

/*
$ node command-builder-fn.js --help
Commands:
  install  Install package

Options:
  --help  Show help                                                    [boolean]

$ node command-builder-fn.js install --help
command-builder-fn.js install

Options:
  --help      Show help                                                [boolean]
*/
```

Bạn cũng có thể cung cấp một handle function để thực hiện xử lý cho command đó. Handle function này được thực thi với object `argv`. Chúng ta thêm phần này cho ví dụ ở trên để in ra package name mà người dùng nhập nhé :D

```javascript
// command-handler.js
require("yargs")
  .command("install", "Install a package", function(yargs) {
    return yargs.option("name", {
      alias: "n",
      describe: "Package name"
    })
  }, function(argv) {
    console.log("Installing package %s...", argv.n);
  })
  .help()
  .argv;

/**
$ node command-handler.js install -n lodash
Installing package lodash...
*/
```

#### Vị trí các tham số

Các command có thể chấp nhận các tham số tùy chọn hay bắt buộc. Nếu là tham số bắt buộc, bạn sử dụng cặp dấu `<>` còn cặp dấu `[]` dành cho tham số tùy chọn.

```javascript
// command-args-positional.js
argv = require("yargs")
  .command("install <package> [global]", "Install package", function(yargs) {
    return yargs.option("global", {
      alias: "g",
      default: false
    })
  }, function(argv) {
    console.log("Install package %s with global option is: %s", argv.package, argv.global);
  })
  .help()
  .argv;

/**
$ node command-args-positional.js install gulp
Install package gulp with global option is: false
$ node command-args-positional.js install gulp -g
Install package gulp with global option is: true
*/
```

Bạn cũng có thể sử dụng dấu `|` để thông báo rằng giá trị của các tham số đó là như nhau.

```javascript
// cmd-args-pos-aliases.js
require("yargs")
  .command("login <email|username> [password]", "Login", {}, function(argv) {
    console.log("Email: %s\nUser: %s\nPassword: %s", argv.email, argv.username, argv.password)
  })
  .help()
  .argv;

/**
$ node cmd-args-pos-aliases.js login namnv 123
Email: namnv
User: namnv
Password: 123
*/
```

Tham số cuối cùng có thể nhận một mảng các giá trị nếu bạn sử dụng cặp dấu `..`:

```javascript
// cmd-args-variadic-pos.js
require("yargs")
  .command("wget <files..>", "Get files", {}, function(argv) {
    console.log("Files: %s", argv.files.join(","));
  })
  .help()
  .argv;

/**
$ node cmd-args-variadic-pos.js wget a.xlsx b.txt c.pdf
Files: a.xlsx,b.txt,c.pdf
*/
```

Bạn cũng có thể chia nhỏ các command thành các file module để tiện quả lý và phát triển. Các file module cần export những thuộc tính sau:

* `exports.command`: Chuỗi hoặc mảng các command.
* `exports.aliases`: Chuỗi hoặc mảng các alias cho các command.
* `exports.describe`: Chuỗi mô tả cho command ở phần help. Sử dụng `false` để tạo command ẩn.
* `exports.builder`: Object chữa các tùy chọn mà command đó chấp nhận hoặc một function trả ra instance của `yargs`.
* `exports.handler`: Function thực hiện công việc của command với các tham số đã được nhập từ command line.

```javascript
// cmds/install.js
exports.command = "install <package> [global]";
exports.describe = "Install a package";
exports.aliases = ["i"];
exports.builder = {
  package: {
    describe: "Package name"
  },
  global: {
    alias: "g",
    describe: "Install package to global",
    default: false
  }
};
exports.handler = function(argv) {
  console.log("Install package %s with global option: %s", argv.package, argv.global);
};

// cmd-module.js
require("yargs")
  // .command("install <package> [global]", "Install a package", require("./cmds/install"))
  .command(require("./cmds/install"))
  .help()
  .argv;

/**
$ node cmd-module.js i lodash
Install package lodash with global option: false
$ node cmd-module.js i lodash -g
Install package lodash with global option: true
*/
```

Cuối cùng cũng đã giới thiệu qua xong phần `.command`. Phần method này có vẻ là khá dài, nhỉ? Cũng đúng thôi, vì nó là phần quan trọng nhất trong việc xây dựng một ứng dụng command line. Giờ chúng ta sẽ chuyển qua method khác nhé.

## `.commandDir(directory, [opts])`

Method này giúp bạn nạp các command từ một thư mục thay vì gọi `.command(require('./dir/module')` nhiều lần. Chúng ta sẽ đi vào chi tiết các tham số mà method này sử dụng nhé:
* `directory`: Thư mục chứa các module command
* `opts`: Object chứa các tùy chọn:
    * `recurse`: Có tìm kiếm trong tất cả các thư mục con hay không. Mặc định là không
    * `extensions`: Mảng giá trị các kiểu mở rộng của file module. Mặc định là `['js']`
    * `visit`: Function sẽ được gọi mỗi khi method `.commandDir` gặp. Function này có 3 tham số bạn có thể sử dụng là `filename`, `commandObject` và `pathToFile`. Function này sẽ trả ra `commandObject` để nhúng vào method `.command`. Bạn có thể sử dụng giá trị `false` để bỏ qua.
    * `include`: Một biểu thức RegExp hay 1 function. Danh sách các module được phép nạp vào command
    * `exclude`: Một biểu thức RegExp hay 1 function. Danh sách các module sẽ không được nạp vào command

Chúng ta thử method này nhé. Cấu trúc thư mục để viết demo cho method này như sau:

```
├── fake-npm.js
├── npm-cmds
│   ├── install.js
│   └── remove.js
```

```javascript
// npm-cmds/install.js
exports.command = "install <packages..>";
exports.describe = "Install packages";
exports.aliases = ["i"];
exports.builder = {
  packages: {
    describe: "Packages name"
  }
};
exports.handler = function(argv) {
  console.log("Install packages %s", argv.packages.join(","));
};

// npm-cmds/remove.js
exports.command = "remove <packages..>";
exports.describe = "Remove package(s)";
exports.aliases = ["rm"];
exports.builder = {
  packages: {
    describe: "Packages name"
  }
};
exports.handler = function(argv) {
  console.log("Remove packages %s", argv.packages.join(","));
};


// fake-npm.js
require("yargs")
  .commandDir("./npm-cmds")
  .help()
  .argv;

/**
$ node fake-npm.js --help
Commands:
  install <packages..>  Install package(s)                          [aliases: i]
  remove <packages..>   Remove package(s)                          [aliases: rm]

Options:
  --help  Show help                                                    [boolean]

$ node fake-npm.js i gulp webpack
Install packages gulp,webpack
$ node fake-npm.js rm gulp webpack
Remove packages gulp,webpack
*/
```

## `.env([Prefix])`

Method này sẽ thông báo với `yargs` rằng cần phải đọc các biến môi trường và áp dụng nó vào `.argv` giống như một tham số với prefix (tiền tố) do bạn quy định.
Nếu method này được gọi mà không có tham số, hoặc tham số là 1 chuỗi rỗng hoặc giá trị là `true` thì tất cả các biến môi trường sẽ được áp dụng vào `.argv`.
Các tham số đưa vào `.argv` được áp dụng theo thứ tự ưu tiên như sau:
1. Tham số do người dùng nhập
2. Config file
3. Biến môi trường
4. Các giá trị mặc định đã được khai báo

Chúng ta cùng xem ví dụ để hiểu hơn về method `.env` này nhé. Mình thử viết lại 2 câu lệnh của Rails là console và server nhé :D!

```javascript
// fake-rails.js

var argv = require("yargs")
  .env("FAKE")
  .option("e", {
    alias: "env",
    default: "development",
    global: true
  })
  .command("console", "Rails console", {}, function(argv) {
    console.log("Start rails console with env %s", argv.env);
  })
  .command("server", "Rails server", {}, function(argv) {
    console.log("Start server with env %s", argv.env)
  })
  .help()
  .argv;

/**
$ node fake-rails.js console
Start rails console with env development
$ node fake-rails.js console -e production
Start rails console with env production
$ FAKE_ENV=production node fake-rails.js console
Start rails console with env production
$ FAKE_ENV=staging node fake-rails.js server
Start server with env staging
*/
```

Đến đây là mình đã giới thiệu sơ qua một số method mà mình thấy quan trọng trong module Yargs. Các bạn có thể đọc chi tiết documentation của nó [**tại đây**](http://yargs.js.org/docs)! Mình xin phép kết thúc series về **Yargs.JS**. Hy vọng qua 2 bài viết mà mình đã chia sẻ sẽ giúp mọi người có cái nhìn cơ bản về việc xây dựng một ứng dụng command line với Node.JS.

Link các ví dụ cho cả 2 phần: [https://github.com/namnv609/viblo-nodejs-yargs-demos](https://github.com/namnv609/viblo-nodejs-yargs-demos)

Chào thân ái và quyết thắng (chao)!
