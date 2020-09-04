---
date: 2015-08-29 07:53:00 +0700
title: PayPal API Service for Laravel
thumbnail: /assets/images/posts/0d2fd56a-8d0a-45ca-86df-dc7793bd1d1b.png
---

Trong dự án hiện tại mình đang tham gia có yêu cầu sử dụng PayPal để thực hiện việc thanh toán. Mình đã tìm hiểu qua nó và dựng thành một service để thành viên trong team có thể sử dụng nó một cách đơn giản nhất có thể<!--more-->. Nay mình xin phép chia sẻ nó với mọi người. Với hy vọng có thể giúp được ai đó khi phải sử dụng PayPal với project của mình :D!

Bắt đầu nhé, mình sẽ hướng dẫn và giải thích chi tiết cho từng bước. Đầu tiên, các bạn cài đặt PayPal PHP SDK bằng composer thông qua câu lệnh:

`sudo composer require paypal/rest-api-sdk-php`

Tiếp theo, chúng ta tạo file config cho PayPal trong thư mục config để cài đặt một số phần quan trọng cho service này.
Tạo file `paypal.php` theo đường dẫn `config/paypal.php`:

```php
<?php
return [
    // Client ID của app mà bạn đã đăng ký trên PayPal Dev
    'client_id' => env('PAYPAL_CLIENT_ID'),
    // Secret của app
    'secret' => env('PAYPAL_SECRET'),
    'settings' => [
        // PayPal mode, sanbox hoặc live
        'mode' => env('PAYPAL_MODE'),
        // Thời gian của một kết nối (tính bằng giây)
        'http.ConnectionTimeOut' => 30,
        // Có ghi log khi xảy ra lỗi
        'log.logEnabled' => true,
        // Đường dẫn đền file sẽ ghi log
        'log.FileName' => storage_path() . '/logs/paypal.log',
        // Kiểu log
        'log.LogLevel' => 'FINE'
    ],
];
```

Tiếp theo, chúng ta sẽ mở file `.env` và thêm ba dòng sau:

```
PAYPAL_CLIENT_ID=<App ID>
PAYPAL_SECRET=<App Secret>
PAYPAL_MODE=<App mode (live or sandbox)>
```

Sau khi cài đặt xong, bạn tạo thư mục Services (nếu chưa có) trong Laravel theo đường dẫn: `app\Services` và tạo file `PayPalService.php` trong đó. Giờ chúng ta sẽ mở file `PayPalService.php` lên và bắt đầu "cột" nhé :D!

Mở đầu mặc định với một file PHP và khai báo namespace cho nó theo đúng đường dẫn của thư mục Services.

```php
<?php

namespace App\Services;
```

Tiếp theo, chúng ta sẽ ``use`` một số class quan trọng của bộ SDK này để có thể tạo transaction, nhận kết quả, lấy danh sách các transaction và hiển thị chi tiết một transaction theo ID.

```php
use PayPal\Rest\ApiContext;
use PayPal\Auth\OAuthTokenCredential;
use PayPal\Api\Item;
use PayPal\Api\ItemList;
use PayPal\Api\Amount;
use PayPal\Api\Transaction;
use PayPal\Api\RedirectUrls;
use PayPal\Api\Payment;
use PayPal\Api\Payer;
use PayPal\Api\PaymentExecution;
use Request;
```

Tiếp theo, chúng ta sẽ khởi tạo class và một số các private property cần thiết nhé. Lý do vì sao mình lại khai báo private vì mình không muốn người khác có thể truy cập trực tiếp vào các property này, mà chỉ muốn cho họ set và get thông qua các getter, setter do mình quy định để đảm bảo cho service được thực hiện đúng.

```php
class PayPalService
{
    // Chứa context của API
    private $apiContext;
    // Chứa danh sách các item (mặt hàng)
    private $itemList;
    // Đơn vị tiền thanh toán
    private $paymentCurrency;
    // Tổng tiền của đơn hàng
    private $totalAmount;
    // Đường dẫn để xử lý một thanh toán thành công
    private $returnUrl;
    // Đường dẫn để xử lý khi người dùng bấm cancel (không thanh toán)
    private $cancelUrl;
```

Tiếp theo, tạo constructor và khai báo một số giá mặc định cho các property:

```php
    public function __construct()
    {
        // Đọc các cài đặt trong file config
        $paypalConfigs = config('paypal');

        // Khởi tạo ngữ cảnh
        $this->apiContext = new ApiContext(
          new OAuthTokenCredential(
              $paypalConfigs['client_id'],
                $paypalConfigs['secret']
            )
        );

        // Set mặc định đơn vị tiền để thanh toán
        $this->paymentCurrency = "USD";

        // Khởi tạo total amount
        $this->totalAmount = 0;
    }
```

Tiếp, chúng ta sẽ viết một loạt các ``getter`` và ``setter`` cho các private property nhé. Đầu tiên sẽ là function đổi đơn vị tiền thanh toán. Với hàm này, bạn nên cẩn thận khi set đơn vị tiền tệ. Ví dụ như khi bạn đổi sang đơn vị tiền Yên của Nhật thì giá của sản phẩm phải là một số nguyên. Nếu là một số thực, bạn sẽ nhận được một ngoại lệ (exception).

```php
    /**
     * Set payment currency
     *
     * @param string $currency String name of currency
     * @return self
     */
    public function setCurrency($currency)
    {
      $this->paymentCurrency = $currency;

        return $this;
    }

    /**
     * Get current payment currency
     *
     * @return string Current payment currency
     */
    public function getCurrency()
    {
        return $this->paymentCurrency;
    }
```

Tiếp theo, đến function thêm item vào list (giống thêm sản phẩm vào giỏ hàng):

```php
    /**
     * Add item to list
     *
     * @param array $itemData Array item data
     * @return self
     */
    public function setItem($itemData)
    {
        // Kiểm tra xem item được thêm vào là một hay một
        // mảng các item. Nếu chỉ là 1 item, thì chúng ta sẽ
        // cho nó thành một mảng item rồi foreach. Việc này giúp
        // chúng ta có thể thêm một hay nhiều item cùng lúc
        if (count($itemData) === count($itemData, COUNT_RECURSIVE)) {
            $itemData = [$itemData];
        }

        // Duyệt danh sách các item
        foreach ($itemData as $data) {
            // Khởi tạo item
            $item = new Item();

            // Set tên của item
            $item->setName($data['name'])
                 ->setCurrency($this->paymentCurrency) // Đơn vị tiền của item
                 ->setSku($data['sku']) // ID của item
                 ->setQuantity($data['quantity']) // Số lượng
                 ->setPrice($data['price']); // Giá
            // Thêm item vào danh sách
            $this->itemList[] = $item;
            // Tính tổng đơn hàng
            $this->totalAmount += $data['price'] * $data['quantity'];
        }

        return $this;
    }

    /**
     * Get list item
     *
     * @return array List item
     */
    public function getItemList()
    {
        return $this->itemList;
    }
```

Tiếp theo đến property ``$totalAmount`` (tổng tiền của đơn hàng). Với property này, chúng ta chỉ viết ``getter`` chứ không viết ``setter``. Vì việc tính tổng tiền của đơn hàng chúng ta đã thực hiện ở trong function ``setItem`` rồi. Với lại nếu cho phép set tổng tiền thì nghe có vẻ hơi vô lý. Ví dụ họ thêm 3 item với giá lần lượt là ``1$``, ``1.5$`` và ``2$`` rồi họ set ``totalAmount`` là ``3$`` thì ngoài việc nó là thảm họa mà chúng ta cũng sẽ nhận được một ngoại lệ (mình cũng không hiểu sao PayPal nó không tính total amount hộ mình mà bắt mình tự tính, thế nhưng khi bạn tính sai, thì nó báo lỗi và bắn exception @@)!

```php
    /**
     * Get total amount
     *
     * @return mixed Total amount
     */
    public function getTotalAmount()
    {
        return $this->totalAmount;
    }
```

Tiếp, chúng ta tạo function để set và get ``$returnUrl`` (đường dẫn để xử lý một thanh toán thành công):

```php
    /**
     * Set return URL
     *
     * @param string $url Return URL for payment process complete
     * @return self
     */
    public function setReturnUrl($url)
    {
        $this->returnUrl = $url;

        return $this;
    }

    /**
     * Get return URL
     *
     * @return string Return URL
     */
    public function getReturnUrl()
    {
        return $this->returnUrl;
    }
```

Đến set và get cho ``$cancelUrl`` để xử lý một thanh toán khi người dùng không muốn thanh toán nữa và nhấn nút Cancel (hủy thanh toán):

```php
    /**
     * Set cancel URL
     *
     * @param $url Cancel URL for payment
     * @return self
     */
    public function setCancelUrl($url)
    {
        $this->cancelUrl = $url;

        return $this;
    }

    /**
     * Get cancel URL of payment
     *
     * @return string Cancel URL
     */
    public function getCancelUrl()
    {
        return $this->cancelUrl;
    }
```

Okie, xong rồi, giờ chúng ta sẽ viết function để tạo transaction. Function này sẽ nhận tất cả các property mà chúng ta đã khai báo ở trên, tạo transaction rồi trả về đường dẫn tương ứng với thanh toán mà chúng ta đã tạo:

```php
    /**
     * Create payment
     *
     * @param string $transactionDescription Description for transaction
     * @return mixed Paypal checkout URL or false
     */
    public function createPayment($transactionDescription)
    {
        $checkoutUrl = false;

        // Chọn kiểu thanh toán.
        $payer = new Payer();
        $payer->setPaymentMethod('paypal');

        // Danh sách các item
        $itemList = new ItemList();
        $itemList->setItems($this->itemList);

        // Tổng tiền và kiểu tiền sẽ sử dụng để thanh toán.
        // Bạn nên đồng nhất kiểu tiền của item và kiểu tiền của đơn hàng
        // tránh trường hợp đơn vị tiền của item là JPY nhưng của đơn hàng
        // lại là USD nhé.
        $amount = new Amount();
        $amount->setCurrency($this->paymentCurrency)
               ->setTotal($this->totalAmount);

        // Transaction
        $transaction = new Transaction();
        $transaction->setAmount($amount)
                    ->setItemList($itemList)
                    ->setDescription($transactionDescription);

        // Đường dẫn để xử lý một thanh toán thành công.
        $redirectUrls = new RedirectUrls();

        // Kiểm tra xem có tồn tại đường dẫn khi người dùng hủy thanh toán
        // hay không. Nếu không, mặc định chúng ta sẽ dùng luôn $redirectUrl
        if (is_null($this->cancelUrl)) {
            $this->cancelUrl = $this->returnUrl;
        }

        $redirectUrls->setReturnUrl($this->returnUrl)
                     ->setCancelUrl($this->cancelUrl);

        // Khởi tạo một payment
        $payment = new Payment();
        $payment->setIntent('Sale')
                ->setPayer($payer)
                ->setRedirectUrls($redirectUrls)
                ->setTransactions([$transaction]);

        // Thực hiện việc tạo payment
        try {
            $payment->create($this->apiContext);
        } catch (\PayPal\Exception\PPConnectionException $paypalException) {
            throw new \Exception($paypalException->getMessage());
        }

        // Nếu việc thanh tạo một payment thành công. Chúng ta sẽ nhận
        // được một danh sách các đường dẫn liên quan đến việc
        // thanh toán trên PayPal
        foreach ($payment->getLinks() as $link) {
            // Duyệt từng link và lấy link nào có rel
            // là approval_url rồi gán nó vào $checkoutUrl
            // để chuyển hướng người dùng đến đó.
            if ($link->getRel() == 'approval_url') {
                $checkoutUrl = $link->getHref();
                // Lưu payment ID vào session để kiểm tra
                // thanh toán ở function khác
                session(['paypal_payment_id' => $payment->getId()]);

                break;
            }
        }

    // Trả về url thanh toán để thực hiện chuyển hướng
        return $checkoutUrl;
    }
```

Đến function kiểm tra trạng thái của một payment dựa theo session có chứa payment ID mà chúng ta đã gán ở function ```createPayment```

```php
    /**
     * Get payment status
     *
     * @return mixed Object payment details or false
     */
    public function getPaymentStatus()
    {
        // Khởi tạo request để lấy một số query trên
        // URL trả về từ PayPal
        $request = Request::all();

        // Lấy Payment ID từ session
        $paymentId = session('paypal_payment_id');
        // Xóa payment ID đã lưu trong session
        session()->forget('paypal_payment_id');

        // Kiểm tra xem URL trả về từ PayPal có chứa
        // các query cần thiết của một thanh toán thành công
        // hay không.
        if (empty($request['PayerID']) || empty($request['token'])) {
            return false;
        }

        // Khởi tạo payment từ Payment ID đã có
        $payment = Payment::get($paymentId, $this->apiContext);

        // Thực thi payment và lấy payment detail
        $paymentExecution = new PaymentExecution();
        $paymentExecution->setPayerId($request['PayerID']);

        $paymentStatus = $payment->execute($paymentExecution, $this->apiContext);

        return $paymentStatus;
    }
```

Nếu thành công. Bạn sẽ nhận được một object chứa thông tin của việc thanh toán như hình dưới:
![paypal_payment_object.png](/assets/images/posts/fe826416-cc96-421b-be1c-4e465104d911.png)

Tiếp theo, đến function lấy danh sách các thanh toán đã được thực hiện. Function nhận 02 tham số là số lượng bản ghi trả về và index của payment muốn lấy (sử dụng để phân trang):

```php
    /**
     * Get payment list
     *
     * @param int $limit Limit number payment
     * @param int $offset Start index payment
     * @return mixed Object payment list
     */
    public function getPaymentList($limit = 10, $offset = 0)
    {
        $params = [
            'count' => $limit,
            'start_index' => $offset
        ];

        try {
            $payments = Payment::all($params, $this->apiContext);
        } catch (\PayPal\Exception\PPConnectionException $paypalException) {
            throw new \Exception($paypalException->getMessage());
        }

        return $payments;
    }
```

Function lấy chi tiết một payment dựa theo payment ID:

```php
    /**
     * Get payment details
     *
     * @param string $paymentId PayPal payment Id
     * @return mixed Object payment details
     */
    public function getPaymentDetails($paymentId)
    {
        try {
            $paymentDetails = Payment::get($paymentId, $this->apiContext);
        } catch (\PayPal\Exception\PPConnectionException $paypalException) {
            throw new \Exception($paypalException->getMessage());
        }

        return $paymentDetails;
    }
```

File test service:

```php
<?php

/**
 * PAYPAL API SERVICE TEST
 */

namespace App\Http\Controllers;

use Auth;
use App\Http\Requests;
use App\Http\Controllers\Controller;
use App\Services\PayPalService as PayPalSvc;

class PayPalTestController extends Controller
{

    private $paypalSvc;

    public function __construct(PayPalSvc $paypalSvc)
    {
        parent::__construct();

        $this->paypalSvc = $paypalSvc;
    }

    public function index()
    {
        $data = [
            [
                'name' => 'Vinataba',
                'quantity' => 1,
                'price' => 1.5,
                'sku' => '1'
            ],
            [
                'name' => 'Marlboro',
                'quantity' => 1,
                'price' => 1.6,
                'sku' => '2'
            ],
            [
                'name' => 'Esse',
                'quantity' => 1,
                'price' => 1.8,
                'sku' => '3'
            ]
        ];
        $transactionDescription = "Tobaco";

        $paypalCheckoutUrl = $this->paypalSvc
                                  // ->setCurrency('eur')
                                  ->setReturnUrl(url('paypal/status'))
                                  // ->setCancelUrl(url('paypal/status'))
                                  ->setItem($data)
                                  // ->setItem($data[0])
                                  // ->setItem($data[1])
                                  ->createPayment($transactionDescription);

        if ($paypalCheckoutUrl) {
            return redirect($paypalCheckoutUrl);
        } else {
            dd(['Error']);
        }
    }

    public function status()
    {
        $paymentStatus = $this->paypalSvc->getPaymentStatus();
        dd($paymentStatus);
    }

    public function paymentList()
    {
        $limit = 10;
        $offset = 0;

        $paymentList = $this->paypalSvc->getPaymentList($limit, $offset);

        dd($paymentList);
    }

    public function paymentDetail($paymentId)
    {
        $paymentDetails = $this->paypalSvc->getPaymentDetails($paymentId);

        dd($paymentDetails);
    }
}

```

Link video demo: [https://drive.google.com/file/d/0B1AQ6cykT8CiTWNMdURmWGlLR2s/view?usp=sharing](https://drive.google.com/file/d/0B1AQ6cykT8CiTWNMdURmWGlLR2s/view?usp=sharing)

Bài viết của mình đến đây là kết thúc rồi :D. Có thể trong bài report tiếp theo, mình sẽ giới thiệu về 02 service còn lại mà mình đã viết cho project hiện tại là gủi mail SendGrid API và resize ảnh Intervention Image.
