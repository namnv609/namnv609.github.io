---
date: 2015-04-24 09:47:00 +0700
title: React.JS with CoffeeScript
---

Như bài giới thiệu về [CoffeeScript](https://viblo.asia/NguyenThiHue/posts/57rVRq84v4bP) của @NguyenThiHue chúng ta biết được nhưng lợi ích của việc viết mã JavaScript bằng CoffeeScript. Và gần đây, React.JS (một JavaScript library mới, do Facebook phát triển) đang ngày càng được sử dụng rộng rãi thì việc viết code React.JS bằng CoffeeScript là một việc đương nhiên mà chúng ta sẽ nghĩ tới. Nhưng khi viết code React.JS, chúng ta luôn phải sử dụng cả JSX (JS + XML) thì lại có vấn đề xảy ra.

```JavaScript
HelloWorld = React.createClass
    render: ->
        (
            <h1>Hello world</h1>
        )
```

Đơn giản như đoạn code trên đây, khi biên dịch chúng ta sẽ nhận được thông báo từ trình biên dịch CoffeeScript như sau:

```Error: Parse error on line 4: Unexpected 'COMPARE'```

Waaath? Chẳng lẽ chúng ta từ bỏ việc kết hợp giữa CoffeeScript với React.JS (JSX) hay sao :(? Lên trang chủ của [CoffeeScript](http://coffeescript.org) xem nó có hỗ trợ ký tự escape không. Oh, may quá, nó có hỗ trợ với ký tự __`__ để báo cho trình biên dịch bỏ qua phần này! OK, bắt đầu thử viết xem nào.
Mình sẽ viết thử đoạn filter bảng dữ liệu nhé:

```JavaScript

### * @jsx React.DOM###

simpleData = [
    name: 'abcd'
    age: 18
,
    name: 'defg'
    age: 23
,
  name: "aijkl"
  age: 20
,
  name: "mnop"
  age: 16
,
  name: "rstu"
  age: 21
,
  name: "vwxy"
  age: 35
]

List = React.createClass
    render: ->
        list = @props.list || {}
        ListElement = `(<h1>No data</h1>)`

        if Object.keys(list).length > 0
            ListElement = list.map (value, key) ->
                `(
                    <p>
                        <span className="name">{value.name}</span>
                        <span className="age">{value.age}</span>
                    </p>
                )`

        `(
            <div>
                <p>
                    <span className="header">Name</span>
                    <span className="header">Age</span>
                </p>
                {ListElement}
            </div>
        )`

Search = React.createClass
    getInitialState: ->
        list: simpleData

    instantSearch: ->
        kw = @refs.kw.getDOMNode().value
        resultList = []

        if kw isnt ""
            simpleData.map (value, key) ->
                if value.name.indexOf(kw) isnt -1 or value.age.toString().indexOf(kw) isnt -1
                    resultList.push value
        else
            resultList = simpleData

        @setState
            list: resultList

    render: ->
        `(
            <div className="search">
                <input type="text" ref="kw" placeholder="Enter name or age" onChange={this.instantSearch} />
                <List list={this.state.list} />
            </div>
        )`

React.renderComponent `<Search />`, document.body

```

Hehe, kiểu viết thân thuộc (với mình) đây rồi. Nhưng với việc sử dụng CoffeeScript không hoàn toàn ổn với JSX. Ví dụ ta có đọa JS sau:

```JavaScript

$.ajax({
    type: 'POST',
    url: '',
    success: function(response) {
        this.setState({
            var: response
        });
    }.bind(this),
    error: function(xhr, ao, err) {
        console.log(err);
    }
});

```

Khi viết bằng CoffeeScript sẽ là:
```JavaScript

$.ajax
    type: 'POST'
    url: ''
    success: (response) ->
        @setState
            var: response
    .bind @
    error: (xhr, ao, err) ->
        console.log err

```

Chúng ta sẽ nhận được lỗi ```Error: Parse error on line 6: Unexpected '.'``` khi biên dịch :(. Mình vẫn chưa tìm được giải pháp tối ưu ngoài việc tạo một bản sao của biến ```this``` trước khi vào callback function:

```JavaScript

_this = @
$.ajax
    type: 'POST'
    url: ''
    success: (response) ->
        _this.setState
            var: response
    error: (xhr, ao, err) ->
        console.log err

```

Mọi người có cách giải quyết hay thì chia sẻ nhé :D!

Source code và demo:

[http://codepen.io/namnv609/pen/NqWeGg](http://codepen.io/namnv609/pen/NqWeGg)

# -Update

Sau khi ngồi ngâm lại code JS được biên dịch ra từ CoffeeScript, mình để ý thấy:
```JavaScript

(function() {

}).call(this);

```
Và mình đã hiểu cách truyền ```this``` vào callback mà không cần phải tạo bản sao của ```this``` trước khi vào callback. Vậy, đoạn JS AJAX ở trên sẽ thành như sau:
```JavaScript

$.ajax
    type: 'POST'
    url: ''
.done(((response) ->
    @setState
        var: response
).bind(@))
.fail (xhr, ao, err) ->
    console.log err

```
Và một ví dụ khác:
```JavaScript

### * @jsx React.DOM ###

Test = React.createClass
    _onClickHandler: (e) ->
        alert e.target.textContent

    render: ->
        list = [0,0,0,0]
        Element = list.map ((value, idx) ->
            `(
                <div onClick={this._onClickHandler.bind(null)}>
                    {idx}: {value}
                </div>
            )`
        ).bind @

        `(
            <div>
                {Element}
            </div>
        )`

React.renderComponent `<Test />`, document.body

```

Vậy là OK rồi. Chắc chỉ còn mỗi việc khi sử dụng __`__ ở ```render``` chúng ta không thể sử dụng ```@``` thay cho ```this``` và cộng chuỗi ```alert "Result is: #{varName}"``` theo kiểu của CoffeeScript được. Nhưng vấn đề đó không là gì so với những gì CoffeeScript mang lại, đúng không ạ :D?

Demo 2:

[http://codepen.io/namnv609/pen/VLwoYZ](http://codepen.io/namnv609/pen/VLwoYZ)
