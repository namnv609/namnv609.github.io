---
date: 2015-07-26 09:05:00 +0700
title: Google Maps API JavaScript Services (Places, Directions, Geocoder)
---

Mình có cơ hội được tìm hiểu về Google Maps API JavaScript V3 từ năm 2011. Nhưng mình không sử dụng nhiều. Do gần đây, dự án mình đang làm có sử dụng đến nó nên mình sẽ viết bài chia sẻ về 3 service mà mình đã tìm hiểu được là Places, Directions và Geocoding.<!--more-->

Về Google Maps API JavaScript thì trên Viblo cũng có khá nhiều bài giới thiệu về nó, ví dụ như [Tìm hiểu về Google Map API](https://viblo.asia/hoang.thi.tuan.dung/posts/z3NVRkwLR9xn) của @hoang.thi.tuan.dung,  bài [TÌM HIỂU VỀ GOOGLE MAP API](https://viblo.asia/nguyenhoa/posts/ZWApGxJ3R06y) của @nguyen.hoa hay bài [Google Maps API](https://viblo.asia/TuyenNguyen/posts/N0bDM621M2X4) của @TuyenNguyen

Các bài viết trên đã giới thiệu khá đầy đủ về Google Maps API, các service và cách tạo một demo đơn giản. Thế nên để tránh mất thời gian, mình sẽ đi thẳng vào demo cho từng thư viện của Google Maps API mà mình đã tìm hiểu được nhé.

## Places Library

### Autocomplete

Đầu tiên, bạn cần require places library bằng lệnh:

```html
<script src="https://maps.googleapis.com/maps/api/js?v=3.exp&sensor=true&libraries=places"></script>
```

Sau đó bạn khởi tạo một div chứa map kèm với một thẻ input để nhận dữ liệu từ người dùng:

```html
<input type="text" id="address" />
<div class="map-canvas"></div>
```

Style tí cho nó đẹp cái nhỉ :D! Mình sẽ dùng SASS nhé.

```sass
#address
    width: 90%
    padding: 4px
    border: 1px solid #000
#map-canvas
    width: 90%
    border: 1px solid #CCC
    height: 400px
    margin: 10px 0
    padding: 4px
```

Tiếp theo, đến phần code JavaScript. Mình sẽ dùng CoffeeScript nhé ^^!

```coffeescript
# Khởi tạo object chứa kinh/vĩ độ. Mình lấy mặc định là Hà Nội nhé :D
mapLatLng = new google.maps.LatLng 21.0277644, 105.83415979999995
# Khởi tạo các tùy chọn cho Map
mapOptions =
    zoom: 12
    center: mapLatLng
# Gắn map vào div
map = new google.maps.Map $("#map-canvas")[0], mapOptions
# Tạo một marker để đánh dấu vị trí hiện tại
marker = new google.maps.Marker
    position: mapLatLng
    map: map
# Lấy input DOM
addressInput = $("#address")[0]
# Khởi tạo Autocomplete
addressAutocomplete = new google.maps.places.Autocomplete addressInput
# Gắn nó vào map
addressAutocomplete.bindTo "bounds", map
# Lắng nghe khi sự kiện địa chỉ được thay đổi. Chúng ta sẽ set cái map và đánh đấu về vị trí đó
google.maps.event.addListener addressAutocomplete, "place_changed", ->
    # Lấy object place
    placeObj = addressAutocomplete.getPlace()
    # Chuyển đánh dấu và map center về vị trí đã nhập
    marker.setPosition placeObj.geometry.location
    map.setCenter placeObj.geometry.location
```

Để lấy được nhiều thứ hay ho hơn (nếu bạn muốn). Bạn có thể `console.log` cái `placeObj` ra để xem nhé ^^! Vậy là xong phần autocomplete rồi. Giờ chúng ta sang phần lấy thông tin các địa điểm xung quanh một vị trí trên map nhé.

### Places search

Places sẽ giúp chúng ta tìm được danh sách các địa điểm (theo từ khóa như **trà đá wifi**, **bún đậu**, **atm vietcombank**, ... hay kiểu của địa điểm như **bus stop**, **hospital**, ..). Places search hỗ trợ chúng ta 04 kiểu tìm kiếm:
* __Nearby search__: Tìm kiếm danh sách các địa điểm xung quanh một vị trí
* __Text search__: Tìm kiếm danh sách các địa điểm thông qua từ khóa
* __Radar search__: Tìm kiếm danh sách các địa điểm theo bán kính nhưng ít thông tin hơn Nearby search và text search
* __Place detail request__: Lấy ra chi tiết của một địa điểm, bao gồm cả các nhận xét của người dùng (từ Google+)

OK, bây giờ mình sẽ làm demo với thằng __nearby search__ cùng với __place detail request__ nhé. Vẫn dùng demo cũ, nhưng chúng ta sẽ thêm một input nữa để nhận từ khóa của người dùng và một nút tìm kiếm nhé:

```html
<input type="text" id="keyword" placeholder="trà đá, coffee, bún đậu, ..." />
<button id="search">Search</button>
```

Thêm tí CSS:

```sass
#keyword
    @extend #address
    margin: 10px 0
#search
    padding: 4px 20px
    border: 1px solid #000
```

Tiếp theo, đến phần JavaScript:

```coffeescript
# Tạo sự kiện khi click search button
$ "#search"
    .on "click", ->
        # Lấy từ khóa người dùng đã nhập
        placeKeyword = $ "#keyword"
            .val().trim()

        # Kiểm tra để chắc chắn từ khóa đã được nhập
        if placeKeyword
            # Khởi tạo object chứa tùy chọn cho việc tìm kiếm địa điểm
            # bao gồm vị trí, bán kính và từ khóa của địa điểm
            placeRequestObj =
                location: mapLatLng
                radius: 5000 # bán kính tính bằng m.
                keyword: placeKeyword

            #Khởi tạo PlacesService
            placesSvc = new google.maps.places.PlacesService map
            # Thực hiện tìm kiếm địa điểm và gọi callback function
            placesSvc.nearbySearch placeRequestObj, searchPlacesResult
        else
            $ "#keyword"
                .focus()

# Callback function nhận kết quả tìm kiếm
searchPlacesResult = (results, status) ->
    # Kiểm tra xem kết quả có thành công không
    if status is google.maps.places.PlacesServiceStatus.OK
        # Vẽ một vòng tròn để đánh dấu bán kính tìm kiếm
        drawCircle = new google.maps.Circle
            map: map
            center: mapLatLng
            radius: 5000
            strokeColor: "#FF0000"
            strokeOpacity: 0.8
            strokeWeight: 1
            strokeColor: "#FF0000"
            fillColor: "#FF0000"
            fillOpacity: 0.35

        # Duyệt mảng kết quả
        i = 0
        while i < results.length
            # Tạo các đánh dấu cho địa điểm trên bản đồ
            placeMarkerMaker results[i]
            i++

# Tạo đánh dấu các điểm và gắn thông tin chi tiết khi click vào các đánh dấu đó
placeMarkerMaker = (place) ->
    # Lấy kinh/vĩ độ của địa điểm
    placeLocation = place.geometry.location
    # Lấy tham chiếu địa điểm để lấy ra thông tin chi tiết
    detailRequest =
        reference: place.reference
    # Lấy chi tiết về địa điểm
    placesSvc.getDetails detailRequest, (details, status) ->
        markerIcon = ''
        if details.icon
            # Tạo icon cho đánh dấu
            markerIcon = new google.maps.MarkerImage details.icon, null, null, null, new google.maps.Size 32, 32
        # Tạo đánh dấu
        placeMarker = new google.maps.Marker
            map: map
            position: placeLocation
            icon: markerIcon
            animation: google.maps.Animation.DROP
        # Gắn sự kiện, khi click vào một địa điểm
        # hiển thị thông tin về địa điểm đó
        google.maps.event.addListener placeMarker, 'click', (e) ->
            infoWindow.setContent "<b>#{details.name}</b><br />#{details.formatted_address}"
            # Bật InfoWindow
            infoWindow.open map, placeMarker
```

Vậy là xong phần tìm kiểm địa điểm bằng PlacesService. Bạn có thể hiển thị chi tiết hơn về địa điểm như số điện thoại, giờ mở/đóng cửa (không sẵn có ở một số nước), hình ảnh, đánh giá, website, ... bằng cách `console.log` biến `details` ra để xem chi tiết nhé ^^!

## Directions service

Tiếp theo, mình sẽ giới thiệu về __Directions service__ của Google Maps API. Service này giúp bạn vẽ đường đi từ điểm A đến điểm B. Chúng ta cũng sẽ làm luôn với demo cũ. Với mục đích khi người dùng click vào một địa điểm, ngoài việc hiển thị chi tiết về địa điểm đó, chúng ta sẽ dẫn đường cho họ luôn, tính từ vị trí hiện tại trên bản đồ.

```coffeescript
# Khởi tạo directions display và directions service
directionsDisplay = new google.maps.DirectionsRenderer
    suppressMarkers: yes # Không hiển thị đánh dấu điểm A và B
directionsSvc = new google.maps.DirectionsService()

# Gắn directions display vào map
# để nó còn biết là sẽ vẽ vào đâu ^^
directionsDisplay.setMap map

# Tạo eventListener cho đánh dấu.
# khi click vào nó, sẽ vẽ đường đi từ vị trí hiện tại trên map
# đến địa điểm được click
google.maps.event.addListener placeMarker, 'click', (e) ->
    # Tạo object chứa tùy chọn cho direction
    directionsRequest =
        origin: mapLatLng # Kinh/vĩ độ điểm A
        destination: e.latLng # Kinh/vĩ độ điểm B (vị trí hiện tại của địa điểm)
        travelMode: google.maps.DirectionsTravelMode.DRIVING # Chọn chế độ lái xe.

    # Set tuyến đường cho directions với tùy chọn ở trên
    directionsSvc.route directionsRequest, (result, status) ->
        # Kiểm tra xem có thể vẽ được đường không
        if status is google.maps.DirectionsStatus.OK
            # Vẽ lên bản đồ với kết quả nhận được
            directionsDisplay.setDirections result
        else
            alert status
```

Ok, xong rồi. Rất đơn giản phải không ạ :D? Phần travelMode, Google hỗ trợ chúng ta 4 kiểu:
* DRIVING: Xe Oto
* WALKING: Đi bộ
* BICYCLING: Xe đạp
* TRANSIT: Xe bus (nếu có)

__Demo chi tiết cho hai service trên (Places, Directions) các bạn có thể xem tại đây__: [http://codepen.io/namnv609/pen/eNPeqp](http://codepen.io/namnv609/pen/eNPeqp)

## Geocoder

Service Geocoder sẽ giúp chuyển đổi một địa chỉ mà chúng ta có thể đọc được hay kinh độ và vĩ độ sang thông tin chi tiết về vị trí đó từ Google Maps. OK, chúng ta thử xem nhé.

Đầu tiên, tạo một input để nhận dữ liệu (có thể là địa chỉ hay kinh/vĩ độ) và một button để thực hiện việc chuyển đổi này:

```html
<input type="text" id="address" />
<button id="detail">Detail</button>
<div id="result"></div>
```

Đến phần JS

```coffeescript
# Gắn sự kiện cho button
$ "#detail"
    .on "click", ->
        # Khởi tạo Geocoder service
        geocoder = new google.maps.Geocoder()
        # Lấy dữ liệu từ người dùng
        address = $ "#address"
            .val().trim()
        # Kiểm tra xem người dùng đã nhập chưa cho chắc
        if address
            # Thực hiện việc chuyển đổi
            geocoder.geocode address: address, (results, status) ->
                # Kiểm tra xem việc chuyển đổi có thành công hay không?
                if status is google.maps.GeocoderStatus.OK
                    ### Ở đây chúng ta sẽ có 1 mảng các object giá trị
                     mà Google sẽ trả về. Các bạn có thể console.log
                     nó ra để xem chi tiết. Ở demo này mình sẽ lấy
                     thông tin của phần tử đầu tiên trong mảng nhé###
                    addressDetail = """
                        Address: #{results[0].formatted_address}
                        Latitude: #{results[0].geometry.location.lat()}
                        Longitude: #{results[0].geometry.location.lng()}
                    """
                    $ "#result"
                        .html addressDetail
                else
                    alert status
        else
            $ "#address"
                .focus()
```

__Demo__: [http://codepen.io/namnv609/pen/rVqKRQ](http://codepen.io/namnv609/pen/rVqKRQ). Các bạn có thể nhập dữ liệu là địa chỉ hoặc lat/lng nhé. Ví dụ:`keangnam me tri, tu liem, ha noi` hoặc `10.8230989, 106.629663` rồi xem kết quả như thế nào nhé ^^!

Trên đây, mình đã giới thiệu qua về những gì mình đã tìm hiểu được từ 3 service của Google Maps API JavaScript V3. Nếu có gì sai sót, mong mọi người góp ý ^^!
