---
date: 2015-09-14 01:38:00 +0700
title: Laravel Services (SendGrid API)
thumbnail: /assets/images/posts/c39d8903-57c5-42dd-927c-1d3fb03da70f.png
---

Như đã nói ở [bài viết trước](/posts/paypal-api-service-for-laravel.html). Hôm nay mình sẽ giới thiệu tiếp service mà mình đã viết trong project vừa qua, đó là gửi mail bằng **SendGrid API**.<!--more-->

## Email service SendGrid API

Trong project vừa qua, có yêu cầu sử dụng email để xác nhận và thông báo một số thứ. Mình đã thử tìm hiểu về service gửi email thông qua SMTP của Laravel. Nhưng mình thấy nó có một số nhược điểm là phải dùng tài khoản email (bao gồm username và password) của nhà cung cấp dịch vụ (như Gmail, Yahoo! Mail, ...). Nếu dùng cá nhân, nó không phải là vấn đề lớn. Nhưng khi giao cho khách, thì việc yêu cầu họ cung cấp cho chúng ta những thông tin như vậy thì không ổn. Vì dự án trước đó mình tham gia cũng có dùng tới **SendGrid** để gửi mail, nên mình quyết định sử dụng nó trong dự án này. Vì **SendGrid** hỗ trợ chúng ta 02 cách để xác thực người dùng, đó là username & password của **SendGrid** hoặc API Key. Nên, việc yêu cầu khách hàng đăng ký ở **SendGrid** và giao cho mình API Key không phải là vấn đề khó khăn vì họ không phải giao những thông tin nhạy cảm như việc giao username và password của email cho mình (honho)!

Okie, giờ chúng ta sẽ đi thẳng luôn vào việc viết service nhé. Đầu tiên bạn cần cài **SendGrid PHP** thông qua **Composer** bằng lệnh:

``sudo composer require sendgrid/sendgrid``

Cài đặt xong, chúng ta sẽ thêm một số cài đặt cho **SendGrid** vào file ``.env``:.

```
SENDGRID_API_KEY=null
SENDGRID_EXCEPTION=true
```

Trong đó, ``SENDGRID_API_KEY`` để chúng ta set API key của **SendGrid**, ``SENDGRID_EXCEPTION`` là để cài đặt việc có xuất exception khi gửi mail bị lỗi hay không. Khi đưa lên production thì bạn nên chuyển thành false cho an toàn nhé.

Tiếp theo, chúng ta sẽ tạo file ``SendGridService.php`` trong thư mục ``app\Services`` nhé. Bắt đầu "cột" nào (ban)!

Mở đầu vẫn là thứ cơ bản của một file PHP thông thường:

```PHP
<?php

namespace App\Services;
```

Chúng ta sẽ use một số class quan trọng để thực hiện việc gửi mail như ``Log`` (để lưu lại exception trong trường hợp xảy ra lỗi) của Laravel và ``SendGrid``, ``Email``, ``Exception`` của **SendGrid**:

```PHP
use Log;
use SendGrid;
use SendGrid\Email as SendGridEmail;
use SendGrid\Exception as SendGridException;
```

Khởi tạo class và khai báo một số property cần thiết:

```PHP
class SendGridService
{
  // Instance của SendGrid
  private $sendGrid;
    // Instance của SendGridEmail
    private $sendGridEmail;
    // Địa chỉ nhận
    private $to;
    // Người gửi
    private $from;
    // Tên người gửi
    private $fromName;
    // Email subject
    private $subject;
    // CC
    private $cc;
    // BCC
    private $bcc;
    // Địa chỉ email khi người dùng click vào Reply
    // Cho trường hợp bạn muốn thay thế $from
    private $replyTo;
    // Dữ liệu mà bạn muốn parse vào view
    private $data;
    // Đường dẫn đến file view
    private $layout;
    // Kiểu email là HTML hay plaintext.
    // SendGrid hỗ trợ gửi cả 02 kiểu cùng lúc, nhưng mình không muốn thế :D
    private $isHtml;
    // Chứa nội dung mail cho trường hợp $isHtml là false
    private $content;
```

Khởi tạo **SendGrid**, **SendGridEmail** instance và giá trị mặc định cho một số property:

```PHP
  public function __construct()
    {
      // Khởi tạo SendGrid instance với các cài đặt từ .env
        $this->sendGrid = new SendGrid(env('SENDGRID_API_KEY', [
          'raise_exceptions' => env('SENDGRID_EXCEPTION', true)
        ]);

        // Khởi tạo SendGridEmail instance
        $this->sendGridEmail = new SendGridEmail();

        // Khởi tạo giá trị mặc định cho các property
        $this->to           = [];
        $this->from         = env('MAIL_FROM');
        $this->fromName     = env('MAIL_NAME');
        $this->cc           = [];
        $this->bcc          = [];
        $this->data         = [];
        $this->isHtml       = true;
        $this->content      = '';
    }
```

Bây giờ, đến phần viết ``setter`` và ``getter`` cho các property. Đầu tiên là property ``$to`` (địa chỉ sẽ nhận email):

```PHP
  /**
     * Set to email address
     *
     * @var mixed $to String or array email address
     * @return App\Services\SendGridService
     */
  public function setTo($to)
    {
      /*
         * Nếu $to là một array chứa danh sách người nhận
         * chúng ta sẽ merge nó lại. Còn là string (một người)
         * thì chúng ta gắn nó vào mảng $to
         */
         if (is_array($to)) {
          $this->to = array_merge($to, $this->to);
         } else {
          $this->to[] = $to;
         }

         return $this;
    }

    /**
     * Get to email address
     *
     * @return array Email address
     */
    public function getTo()
    {
        return $this->to;
    }
```

Đến property ``$from`` và ``$fromName``. Vì ở ``__constructor``, chúng ta lấy ``$from`` và ``$fromName`` từ file ``.env`` nên chúng ta chỉ có ``getter`` chứ không có ``setter``:

```PHP
  /**
     * Get email send from
     *
     * @return string Email address
     */
    public function getFrom()
    {
        return $this->from;
    }

    /**
     * Get Email from name
     *
     * @return string From name
     */
    public function getFromName()
    {
        return $this->fromName;
    }
```

Đến 02 property ``$cc`` (carbon copy) gửi bản sao đến địa chỉ email này và ``$bcc`` (blind carbon copy) cũng giống như CC nhưng khác CC ở chỗ là  người nhận sẽ không biết chúng ta đã BCC đến những địa chỉ email nào. Vì 02 property này cũng nhận ``string`` hoặc ``array`` giống ``$to`` nên mọi người có thể scroll ngược lên để xem ở function ``setTo`` nhé.

```PHP
  /**
     * Set CC
     *
     * @var mixed $cc String or array email address
     * @return App\Services\SendGridService
     */

    public function setCC($cc)
    {
        if (is_array($cc)) {
            $this->cc = array_merge($cc, $this->cc);
        } else {
            $this->cc[] = $cc;
        }

        return $this;
    }

    /**
     * Get CC
     *
     * @return array CC email address
     */
    public function getCC()
    {
        return $this->cc;
    }

    /**
     * Set BCC
     *
     * @param mixed $bcc String or array email address
     * @return App\Services\SendGridService
     */
    public function setBCC($bcc)
    {
        if (is_array($bcc)) {
            $this->bcc = array_merge($bcc, $this->bcc);
        } else {
            $this->bcc[] = $bcc;
        }

        return $this;
    }

    /**
     * Get BCC
     *
     * @return array BCC email address
     */
    public function getBCC()
    {
        return $this->bcc;
    }
```

Tiếp theo là property ``$replyTo``. Property này mình chỉ nhận một địa chỉ email sẽ được reply nên không cần phải kiểm tra như 03 thằng ``$to``, ``$cc`` và ``$bcc``:

```PHP
  /**
     * Set reply to
     *
     * @param string $replyTo Reply to email address
     * @return App\Services\SendGridService
     */
    public function setReplyTo($replyTo)
    {
        $this->replyTo = $replyTo;

        return $this;
    }

    /**
     * Get reply to address
     *
     * @return string Reply to email address
     */
    public function getReplyTo()
    {
        return $this->replyTo;
    }
```

Đến tiêu đề của email:

```PHP
  /**
     * Set email subject
     *
     * @param string $subject Email subject
     * @return App\Services\SendGridService
     */
    public function setSubject($subject)
    {
        $this->subject = $subject;

        return $this;
    }

    /**
     * Get email subject
     *
     * @return string Email subject
     */
    public function getSubject()
    {
        return $this->subject;
    }
```

Data (dữ liệu) sẽ được parse vào view trong trường hợp gửi HTML:

```PHP
  /**
     * Set email layout variable data
     *
     * @var array $data Email data
     * @return App\Services\SendGridService
     */
    public function setData($data)
    {
        $this->data = $data;

        return $this;
    }

    /**
     * Get email data
     *
     * @return array Email layout variable data
     */
    public function getData()
    {
        return $this->data;
    }
```

Đường dẫn đến file template chứa HTML:

```PHP
  /**
     * Set email layout (template)
     *
     * @param string $layoutPath Path to email layout
     * @return App\Services\SendGridService
     */
    public function setLayout($layoutPath)
    {
        $this->layout = $layoutPath;

        return $this;
    }

    /**
     * Get email template layout
     *
     * @return string Path to email template layout
     */
    public function getLayout()
    {
        return $this->layout;
    }
```

Kiểu gửi email (HTML hoặc plaintext):

```PHP
  /**
     * Set email type (HTML or Text only)
     *
     * @param bool $isHtml True or false
     * @return App\Services\SendGridService
     */
    public function setIsHtml($isHtml)
    {
        $this->isHtml = $isHtml;

        return $this;
    }

    /**
     * Get email type
     *
     * @return bool Email type is HTML?
     */
    public function getIsHtml()
    {
        return $this->isHtml;
    }
```

Nội dung cho email không phải là HTML (plaintext):

```PHP
  /**
     * Set email content (for email type isHtml is false)
     *
     * @param string Email content
     * @return App\Services\SendGridService
     */
    public function setContent($content)
    {
        $this->content = $content;

        return $this;
    }
```

Lấy nội dung email (cho cả 02 trường hợp là HTML hoặc plaintext):

```PHP
  /**
     * Get email content (for both HTML and Text)
     *
     * @return string Email content (HTML or Text)
     */
    public function getContent()
    {
        $content = $this->content;

        if ($this->isHtml) {
            $content = view($this->layout, $this->data)->render();
        }

        return $content;
    }
```

Cuối cùng là function gửi mail nữa là chúng ta đã hoàn thành service này (hura):

```PHP
  /**
     * Send email
     *
     * @return bool Send email status
     */
    public function send()
    {
      // Lấy nội dung email
        $content = $this->getContent();

        // Set một số thông tin cơ bản cho SendGridEmail
        $this->sendGridEmail->setTos($this->to)
                            ->setFrom($this->from)
                            ->setFromName($this->fromName)
                            ->setSubject($this->subject);

    // Set nội dung email cho trường hợp HTML hoặc plaintext
        if ($this->isHtml) {
            $this->sendGridEmail->setHtml($content);
        } else {
            $this->sendGridEmail->setText($content);
        }

      // Kiểm tra một số optional property, nếu nó có thì mới set
        if (count($this->cc)) {
            $this->sendGridEmail->setCcs($this->cc);
        }

        if (count($this->bcc)) {
            $this->sendGridEmail->setBccs($this->bcc);
        }

        if ($this->replyTo) {
            $this->sendGridEmail->setReplyTo($this->replyTo);
        }

        // OK, gửi mail nào!
        try {
            $this->sendGrid->send($this->sendGridEmail);

            // Nếu thành công, trả về true
            return true;
        } catch (SendGridException $e) {
          // Bẫy SendGrid exception
            Log::error('SendGrid error code: ' . $e->getCode());
            Log::error($e->getErrors());
        } catch (\Exception $e) {
          // Đặt bẫy tiếp, phòng khi không phải lỗi của thằng SendGrid :D
            Log::error($e->getMessage());
        }

        // Còn không gửi được vì lý do nào đó, vào log xem.
        // Còn việc của chúng ta là trả về false.
        return false;
    }
```

Vậy là xong service gửi mail bằng **SendGrid API** rồi. Trong bài viết tiếp theo mình sẽ giới thiệu nốt service cuối cùng mà mình đã viết trong dự án vừa qua, đó là resize ảnh bằng **Intervention Image**. Do điều kiện không cho phép để có demo trực tiếp. Nên mình sẽ dùng video demo như bài viết giới thiệu PayPal API service nhé :D!

Link video demo: [https://drive.google.com/file/d/0B1AQ6cykT8CiYUFBRXc2aTdNSWs/view?usp=sharing](https://drive.google.com/file/d/0B1AQ6cykT8CiYUFBRXc2aTdNSWs/view?usp=sharing)
