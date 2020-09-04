---
date: 2015-10-07 01:55:00 +0700
title: Laravel image service with Intervention Image
thumbnail: /assets/images/posts/0e2d5053-be75-49c2-b3b5-c316649c9351.png
---

Tiếp tục series về Laravel service mà mình đã viết trong dự án mình đã tham gia. Resize ảnh bằng [**Intervention Image**](http://image.intervention.io/)!<!--more-->

Dự án đó bọn mình làm về các tour du lịch. Nên việc sử dụng hình ảnh để giới thiệu là không thể thiếu. Ngoài ra, những hình ảnh được sử dụng trong trang (do người dùng đưa lên) phải là ảnh độ phân dải cao, dung lượng lớn và khá nhiều cho một tour.

Việc sử dụng ảnh thumbnail cho các ảnh lớn đấy khi người dùng chưa click vào là tất nhiên. Nhưng khi dev, mọi người chỉ tạo dữ liệu giả để làm, nên không chú ý đến vấn đề dung lượng ảnh và tốc độ tải trang (do làm việc ở local). Nên mọi người resize ảnh bằng cách đặt thuộc tính width, height chứ không quan tâm đến dung lượng thật sự.

Khi đưa lên staging server thì mọi chuyện vẫn chưa có gì cả. Nhưng khi cho lên production server, dữ liệu thật được đưa lên thì mình đã nhận ra rằng cách hình ảnh họ đưa lên là ảnh phân giải lớn, dung lượng cao, nên việc browser load và render từng phần ảnh là điều rất bất cập.

Mình đã tìm kiếm và gặp thằng [**Intervention Image**](http://image.intervention.io/). Đọc qua documentation thì nó hợp với yêu cầu của mình. Việc mình phải làm tiếp theo là viết nó thành một service đơn giản để mọi người trong team có thể dùng lại và dùng ở nhiều nơi. Okie, đi vào chi tiết cái service này nhé :D!

Đầu tiên, cài đặt nó bằng **Composer** thông qua câu lệnh:

```
sudo composer require intervention/image
```

Sau đó, vẫn là tạo một file php trong thư mục `app/Services` như 02 cái trước. Mình dùng tên `ThumbnailService.php`. Tiếp theo, bắt đầu code những dòng căn bản như khai báo `namespace`, `class name` và `use` thư viện Intervention Image:

```php
<?php

namespace App\Services;

use Intervention\Image\ImageManager;

class ThumbnailService
{
```

Khai báo một số ``property`` sẽ dùng:

```php
    /**
     * Instance của Intervention\Image\ImageManager
     */
    private $imageManager;

    /**
     * Đường dẫn đến file sẽ resize
     */
    private $imagePath;

    /**
     * Tỷ lệ ảnh.
     * Dành cho trường hợp chỉ sử dụng width của ảnh thumbnail
     */
    private $thumbRate;

    /**
     * Width của ảnh thumb
     */
    private $thumbWidth;

    /**
     * Height của ảnh thumb
     */
    private $thumbHeight;

    /**
     * Thư mục sẽ chứa ảnh đã được resize
     */
    private $destPath;

    /**
     * Tọa độ X. Cho trường hợp crop ảnh
     */
    private $xCoordinate;

    /**
     * Tọa độ Y. Cho crop ảnh
     */
    private $yCoordinate;

    /**
     * Vị trí sẽ dùng cho cả 2 trường hợp crop và resize. Là fit
     */
    private $fitPosition;

    /**
     * Tên ảnh thumb sẽ được lưu
     */
    private $fileName;
```

Khởi tạo giá trị mặc định cho một số property.

```php
    public function __construct()
    {
        /**
         * Khởi tạo instance của Intervention Image.
         * Hỗ trợ 2 image extension của PHP. là Imagik và GD
         * Mình dùng GD.
         */
        $this->imageManager = new ImageManager([
            'driver' => 'gd'
        ]);

        /**
         * Tỷ lệ ảnh.
         * Mặc định sẽ là tỉ lệ 3/4 (1024x768, 800x600, ..)
         */
        $this->thumbRate = 0.75;
        // Tọa độ X
        $this->xCoordinate = null;
        // Tọa độ Y
        $this->yCoordinate = null;
        // Vị trí sẽ dùng để crop và resize
        $this->fitPosition = 'center';
    }
```

Tiếp đến, viết `getter` và `setter` cho các `property`. Đầu tiên là `$imagePath`

```php
    /**
     * @param string $imagePath Đường dẫn đến ảnh cần resize
     * @return App\Services\ThumbnailService
     */
    public function setImage($imagePath)
    {
        $this->imagePath = $imagePath;

        return $this;
    }

    /**
     * @return string $imagePath
     */
    public function getImage()
    {
        return $this->imagePath;
    }
```

Tỷ lệ ảnh `$thumbRate`:

```php
    /**
     * @param double Tỷ lệ ảnh sẽ resize
     * @return App\Services\ThumbnailService
     */
    public function setRate($rate)
    {
        $this->thumbRate = $rate;

        return $this;
    }

    /**
     * @return double $thumbRate
     */
    public function getRate()
    {
        return $this->thumbRate;
    }
```

Size ảnh `$thumbWidth` và `$thumbHeight`:

```php
    /**
     * @param integer $thumbWidth
     * @param integer $thumbHeight
     * @return App\Services\ThumbnailService
     */
    public function setSize($width, $height = null)
    {
        $this->thumbWidth = $width;
        $this->thumbHeight = $height;

        /**
        * Nếu $height là null thì dùng tỉ lệ ảnh
        */
        if (is_null($height)) {
            $this->thumbHeight = ($this->thumbWidth * $this->thumbRate);
        }

        return $this;
    }

    /**
     * @return array Mảng chứa $thumbWidth và $thumbHeight
     */
    public function getSize()
    {
        return [$this->thumbWidth, $this->thumbHeight];
    }
```

Đường dẫn sẽ lưu ảnh đã resize `$destPath`:

```php
    /**
     * @param string $destPath Đường dẫn sẽ lưu ảnh
     * @return App\Services\ThumbnailService
     */
    public function setDestPath($destPath)
    {
        $this->destPath = $destPath;

        return $this;
    }

    /**
     * @return string $destPath
     */
    public function getDestPath()
    {
        return $this->destPath;
    }
```

Tọa độ X và Y để dùng trong trường hợp crop ảnh. Tính từ góc trên cùng bên trái:

```php
    /**
     * @param integer $xCoord Tọa độ X
     * @param integer $yCoord Tọa độ Y
     * @return App\Services\ThumbnailService
     */
    public function setCoordinates($xCoord, $yCoord)
    {
        $this->xCoordinate = $xCoord;
        $this->yCoordinate = $yCoord;

        return $this;
    }

    /**
     * @return array Mảng tọa độ X-Y
     */
    public function getCoordinates()
    {
        return [$this->xCoordinate, $this->yCoordinate];
    }
```

Vị trí để sử dụng khi fit ảnh (resize và crop). Các vị trí được hỗ trợ bao gồm:
    * top-left
    * top
    * top-right
    * left
    * center (mặc định)
    * right
    * bottom-right
    * bottom
    * bottom-left

```php
    /**
     * @param string Vị trí dùng để fit
     * @return App\Services\ThumbnailService
     */
    public function setFitPosition($position)
    {
        $this->fitPosition = $position;

        return $this;
    }

    /**
     * @return string $fitPosition
     */
    public function getFitPosition()
    {
        return $this->fitPosition;
    }
```

Tên file ảnh sau khi resize:

```php
    /**
     * @param string Tên file sẽ lưu sau khi resize
     * @return App\Services\ThumbnailService
     */
    public function setFileName($fileName)
    {
        $this->fileName = $fileName;

        return $this;
    }

    /**
     * @return string $fileName
     */
    public function getFileName()
    {
        return $this->fileName;
    }
```

Vậy là xong `getter` và `setter` cho các `property`. Giờ chúng ta sẽ đi vào trọng tâm chính của cái service này nhé. Phần thực hiện resize (crop, fit) ảnh :D!

```php
    /**
     * @param string $type Kiểu ảnh thumb. fit, crop hoặc resize
     * @param integer $quality Chất lượng ảnh thumbnail
     * @return mixed Tên file đã resize hoặc false khi xảy ra lỗi
     */
    public function save($type = 'fit', $quality = 80)
    {
        // Lấy tên file sẽ lưu từ file sẽ resize
        $fileName = pathinfo($this->imagePath, PATHINFO_BASENAME);

        /**
         * Nếu property $this->fileName không null (đã được set)
         * Sử dụng nó :D
         */
        if ($this->fileName) {
            $fileName = $this->fileName;
        }

        // Ghép $this->destPath và $fileName lại để có được vị trí file thumb sẽ được lưu
        $destPath = sprintf('%s/%s', trim($this->destPath, '/'), $fileName);

        /**
         * Tạo đối tượng ảnh từ Intervention Image Manage
         * Với đối tượng này, chúng ta có thể thao tác được hầu hết
         * các function mà Intervention Image hỗ trợ
         * Chi tiết các bạn có thể vào trang chủ của nó để xem
         */
        $thumbImage = $this->imageManager->make($this->imagePath);

        /**
         * Kiểm tra kiểu ảnh thumb được dùng. Mặc định sẽ là fit
         * Mỗi kiểu sẽ sử dụng các tham số phù hợp
         */
        switch ($type) {
            case 'resize':
                $thumbImage->resize($this->thumbWidth, $this->thumbHeight);
                break;
            case 'crop':
                $thumbImage->crop($this->thumbWidth, $this->thumbHeight, $this->xCoordinate, $this->yCoordinate);
                break;
            default:
                $thumbImage->fit($this->thumbWidth, $this->thumbHeight, null, $this->fitPosition);
        }

        // Đặt bẫy cho chắc :D
        try {
            // Lưu xuống disk
            $thumbImage->save($destPath, $quality);
        } catch (\Exception $e) {
            // Log lại lỗi rồi trả false
            \Log::error($e->getMessage());

            return false;
        }

        // Lưu thành công rồi. Trả về đường dẫn tới ảnh đã lưu
        return $destPath;
    }
```

Basic usage:

```php
use App\Services\ThumbnailService;

$thumbSvc = new ThumbnailService();

// Resize
$thumbSvc->setImage('path/to/image/')
         ->setSize(1024)
         ->setDestPath('/path/to/destination')
         ->save('resize');

// Crop
$thumbSvc->setImage('path/to/image/')
         ->setSize(1024)
         ->setDestPath('/path/to/destination')
         ->setCoordinates(10, 20)
         ->save('crop');

// Fit
$thumbSvc->setImage('path/to/image/')
         ->setSize(1024)
         ->setDestPath('/path/to/destination')
         ->setFitPosition('top-left')
         ->save();

// Custom file name
$thumbSvc->setImage('path/to/image/')
         ->setSize(1024)
         ->setDestPath('/path/to/destination')
         ->setFileName(md5(time()) . '.extension')
         ->save('resize');
```

Vậy là xong cái service đơn giản để resize ảnh rồi :D! Tiện đây, mình xin giới thiệu component mình viết cho Laravel 5 từ thằng [**Intervention Image**](http://image.intervention.io/) này. Đó là resize ảnh theo URL (on the fly). Về căn bản thì nó như thằng [**TimThumb**](http://www.binarymoon.co.uk/projects/timthumb/). Nếu ai có nhã hứng thì giúp mình viết một bài chia sẻ, hướng dẫn cài đặt và sử dụng bằng tiếng Anh nhé. Mình kém khoản viết documentation lắm. Mình muốn có người sử dụng để phát hiện ra lỗi và phát triển thêm cái component này :D!
* Link Github của component: [https://github.com/namnv609/l5-fly-thumb](https://github.com/namnv609/l5-fly-thumb)
* Link video hướng dẫn cài đặt và sử dụng: [https://drive.google.com/file/d/0B1AQ6cykT8CiMjNzdVp3cTVQN0k/view?usp=sharing](https://drive.google.com/file/d/0B1AQ6cykT8CiMjNzdVp3cTVQN0k/view?usp=sharing)

Bài tiếp theo mình sẽ giới thiệu về Repository design pattern trong Laravel nhé.
