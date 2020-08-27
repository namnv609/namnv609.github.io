---
date: 2015-05-14 09:23:00 +0700
title: Yahoo! Query Language (YQL)
---

Yahoo! Query Language (YQL) được tạo bởi Yahoo!, nó là một ngôn ngữ truy vấn giống với SQL. Nó cho phép chúng ta truy vấn, lọc và kết hợp dữ liệu giữa các website với nhau thông qua một ngôn ngữ đơn giản nhất.<!--more-->

__Các tính năng tiêu biểu của YQL:__
* _Truy cập dữ liệu thông qua web_
    * Chọn, lọc, sắp xếp và nối dữ liệu qua các dịch vụ web. Thậm chí bạn có thể thêm, sửa và xóa từ YQL.
* _Tăng tốc độ thực thi ứng dụng_
    * Với YQL, các ứng dụng chạy nhanh hơn với ít dòng code và một mạng lưới đánh dấu (footprint) nhỏ hơn.
* _Dễ dàng khai thác HTML_
    * Khai thác HTML từ các website để biến nó thành dữ liệu để tái sử dụng. Tạo API từ các website không hỗ trợ API (non-official API).
* _Ghép các nguồn dữ liệu_
    * Trộn và kết hợp dữ liệu từ các nguồn khác nhau bằng cách sử dụng YQL sub-selects.
* _Chuyển đổi XML sang JSON_
    * YQL có thể chuyển đổi XML sang JSON và ngược lại. Truy cập Atom, RSS hoặc nhiều hơn thế. Thậm chí bạn có thể tải các tập tin CSV từ bất kỳ đâu.
* _Tính mở rộng_
    * Định nghĩa các bảng dữ liệu mở (Open data tables) để truy cập mọi nguồn dữ liệu khác Yahoo Web Services

Để bắt đầu với YQL, các bạn có thể truy cập vào [YQL Console](https://developer.yahoo.com/yql/console/) để xem và test các ví dụ chi tiết hơn. Còn bài viết này mình chỉ giới thiệu qua về YQL và cái mình chú trọng vào nhất sẽ ở phần **Tạo API từ các website không hỗ trợ API** nhé :D!

Mình sẽ hướng dẫn mọi người viết một non-official API đơn giản bằng JavaScript (mình sẽ viết bằng CoffeeScript, còn code JS thì mọi người vào link source code ở cuối bài để xem phần compiled) nhé. Trong bài này, mình sẽ sử dụng trang [NhacCuaTui](http://nhaccuatui.com) để làm ví dụ. API này có hai việc chính là nhận từ khóa (tên bài hát, ca sỹ, người upload, ...) rồi trả về danh sách các bài hát tương ứng và nhận đường dẫn bài hát rồi trả về link trực tiếp (direct link) của bài hát đó. Okie, let's go!

Chúng ta tạo hẳn một class với tên NhacCuaTui cho hịn nhé :))!

```JavaScript
NhacCuaTui = ->
```

Trong class này có một private object chứa hai phần tử là URL của YQL và URL tìm kiếm của NhacCuaTui (mình sử dụng domain dành cho mobile để có tốc độ truy xuất nhanh hơn):

```JavaScript
    _apiSettings =
        apiURL: "http://m.nhaccuatui.com/tim-kiem/bai-hat?q="
        endpointURL: "https://query.yahooapis.com/v1/public/yql?q="
```

Tiếp theo, chúng ta sẽ có một private function để truy xuất dữ liệu từ YQL, hàm này sẽ nhận câu truy vấn và trả về object dữ liệu từ YQL:

```JavaScript
    _yqlExecuteQuery = (yqlStatement) ->
        yqlResult = {}
        $.ajax
            url: "#{_apiSettings.endpointURL}#{encodeURIComponent yqlStatement}&format=json"
            cache: no
            async: no
            success: (response) ->
                yqlResult = response
            error: (xmlHttpReq, ajaxOpts, error) ->
                console.log error

        yqlResult
```

Tiếp theo đến public function search, nhận tham số là từ khóa và trả về một object chứa danh sách kết quả tìm được:

```JavaScript
    @search = (keyword) ->
        # Mã hóa từ khóa sang dạng URI #
        keyword = encodeURIComponent keyword.trim()
        # Object chứa status và mảng object kết quả #
        result =
            status: false
            items: []

        if keyword isnt ""
            yqlStatement = "SELECT * FROM html WHERE url='#{_apiSettings.apiURL + keyword}' AND xpath='//div[contains(@class, \"bgmusic\")]'"
            yqlResult = _yqlExecuteQuery yqlStatement

            if yqlResult.query.count > 0
                result.status = true
                $.each yqlResult.query.results.div, (idx, item) ->
                    result.items.push
                        title: item.h3.a.title
                        link: item.h3.a.href
                        artist: item.p.content
                        listen: item.p.span.content

        result
```

Xong phần tìm kiếm. Bây giờ đến phần lấy direct link của bài hát. Public function này sẽ nhận tham số là link bài hát và trả về object chứa trạng thái với direct link của bài hát đó:

```JavaScript
    @get = (link) ->
        result =
            status: false
            link: ""

        if link.trim() isnt ""
            yqlStatement = "SELECT * FROM html WHERE url='#{link}' AND xpath='//div[@class=\"download\"]//a'"
            yqlResult = _yqlExecuteQuery yqlStatement

            if yqlResult.query.count > 0
                result =
                    status: true
                    link: yqlResult.query.results.a.href

        result
```

Vậy là xong rồi. Bây giờ chúng ta sẽ thử test API của mình nhé. Mình sẽ sử dụng React.JS để test. Từ khi quen viết React.JS mình rất ngại sửa HTML DOM bằng jQuery :D!

```JavaScript
###* @jsx React.DOM ###

SearchBar = React.createClass
    _onClickSearchButtonHandler: ->
        keyword = @refs.keyword.getDOMNode().value.trim()

        if keyword isnt ""
            @props.searchSong keyword
        else
            alert "Please enter keyword."

    render: ->
        # Trong source và demo không có dấu ` ` trước và sau thẻ input và button nhé #
        `(
            <div className="search-bar">
                `<input type="text" placeholder="Song name, artist, user, ..." ref="keyword" />`
                `<button onClick={this._onClickSearchButtonHandler}>Search</button>`
            </div>
        )`

SongList = React.createClass
    render: ->
        songList = @props.songList || []

        SongElement = `(<div className="song">
                            <p className="title">(no data)</p>
                        </div>)`

        if songList.length > 0
            SongElement = songList.map ((song, idx) ->
                `(<Song songInfo={song} getSong={this.props.getSong} />)`
            ).bind @
        `(
            <div className="song-list">
                {SongElement}
            </div>
        )`

Song = React.createClass
    _onClickSongHandler: (link) ->
        @props.getSong link

    render: ->
        songInfo = @props.songInfo

        `(
            <div className="song" onClick={this._onClickSongHandler.bind(null, songInfo.link)}>
                <p className="title">{songInfo.title}</p>
                <p>
                    <span className="artist">{songInfo.artist}</span>
                    <span className="listen">{songInfo.listen}</span>
                </p>
            </div>
        )`

Player = React.createClass
    componentDidUpdate: (prevProps, prevState) ->
        videoElement = @refs.videoElement.getDOMNode()
        videoElement.load()
        videoElement.play()

    render: ->
        songLink = @props.link || ""

        `(
            <div id="player">
                <audio controls autoplay ref="videoElement">
                    <source src={songLink} type="audio/mpeg" />
                    Your browser does not support the audio element.
                </audio>
            </div>
        )`

App = React.createClass
    getInitialState: ->
        songList: []
        songLink: ""

    componentWillMount: ->
        @nhacCuaTui = new NhacCuaTui()

    _searchSong: (keyword) ->
        songList = @nhacCuaTui.search keyword

        if songList.status
            @setState
                songList: songList.items

    _getSong: (link) ->
        songLink = @nhacCuaTui.get link

        if songLink.status
            @setState
                songLink: songLink.link

    render: ->
        `(
            <div id="wrapper">
                <SearchBar searchSong={this._searchSong} />
                <SongList songList={this.state.songList} getSong={this._getSong} />
                <Player link={this.state.songLink} />
            </div>
        )`

React.renderComponent `<App />`, document.body
```

Các bạn có thể xem demo và source code tại hai địa chỉ sau:

* Demo: [http://codepen.io/namnv609/pen/mJPLxa](http://codepen.io/namnv609/pen/mJPLxa)
* Source: [https://github.com/namnv609/test-nhaccuatui-api](https://github.com/namnv609/test-nhaccuatui-api)

Trên đây mình đã giới thiệu với mọi người về YQL và cũng như cách để viết một API của riêng mình. Trước khi viết, mọi người nên quan tâm đến việc bản quyền nội dung nhé. Demo và source của mình chỉ có mục đích để giới thiệu thôi nhé :D!
