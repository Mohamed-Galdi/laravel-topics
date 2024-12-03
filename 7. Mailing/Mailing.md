# Table of Contents

1. [Configuration](#1-configuration)
    - [a- RESEND integration example](#a--resend-integration-example)

2. [Mailables](#2-mailables)

3. [Email Sender](#3-email-sender)
    - [a- Using the Envelope](#a--using-the-envelope)
    - [b- Using a Global `from` Address](#b--using-a-globalfromaddress)

4. [Email Content](#4-email-content)
    - [a- Markdown messages](#a--markdown-messages)

5. [Pass Data To View](#5-pass-data-to-view)
    - [a- Public Properties](#a--public-properties)
    - [b- with Parameter](#b--with-parameter)

6. [Sending Emails](#6-sending-emails)

7. [Rendering Mailables](#7-rendering-mailables)

8. [Events](#8-events)

9. [Queueing Mail](#9-queueing-mail)


Email communication is a crucial part of modern web applications. Whether it’s sending notifications, password reset links, or newsletters, Laravel makes it a breeze to integrate email functionality into your project.

# 1. Configuration

Laravel provides drivers for sending emails via SMTP, Mailgun, Postmark, Resend, and Amazon SES, allowing you to integrate email services seamlessly with your application. These services can be configured through the application's `config/mail.php` file.

You can configure your application to use different email services for specific types of emails. For instance:

- **Postmark** for transactional emails

- **Amazon SES** for bulk emails

Within the `mail` configuration file, you will find a `mailers` array. This array contains sample configurations for each major mail driver supported by Laravel. The `default` configuration value specifies which mailer will be used by default for sending emails.

Laravel supports various drivers for sending emails:

- **SMTP** (Traditional email servers)

- **API-based drivers** (Mailgun, Postmark, Resend, etc.)

> **Note:** API-based drivers like Mailgun and Postmark are often simpler and faster than traditional SMTP servers.

Below is a sample of the `config/mail.php` configuration file:

```php
[
    'default' => env('MAIL_MAILER', 'log'),

    'mailers' => [

        'smtp' => [
            // ...    
        ],

        'ses' => [
            // ...
        ],

        'postmark' => [
            // ...
        ],

        'resend' => [
            // ...
        ],

    ],

    'from' => [
        'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
        'name' => env('MAIL_FROM_NAME', 'Example'),
    ],

];
```

#### a- RESEND integration example:

Resend is one of the best driver options for email delivery. It is easy to integrate, offers 3,000 free emails monthly, and provides an elegant and modern interface.

Start by installing Resend's PHP or Laravel SDK via Composer:

```shell
composer require resend/resend-php
# or
composer require resend/resend-laravel
```

Next, set the `default` option in your application's `config/mail.php` configuration file to `resend`.

```php
// config/mail.php
'resend' => [
 'key' => env('RESEND_KEY'),
],
```

Then ensure that your `config/services.php` configuration file contains the following options:

```php
// config/services.php
'resend' => [
    'key' => env('RESEND_KEY'),
],
```

Last ket your **RESEND_API_KEY** from resend dashboard and save it on your **.env** file:

```php
// .env
RESEND_API_KEY=re_123abc
MAIL_MAILER=resend
MAIL_FROM_ADDRESS=onboarding@resend.dev // you can use your custom domain
MAIL_FROM_NAME="${APP_NAME}"
```

**Domain verefication**:

Resend simplifies domain verification. To verify a domain you own, add the required DNS records provided by Resend to your domain settings. Once the records are approved, you can send emails from your custom domain.

> **Tip:** Ensure your DNS records are correctly configured to avoid delays in verification. 

# 2. Mailables

A mailables is a class that represent a type of emails sent by your application, for example we can have a mailable for *password resseting* and another for *order shipping* and so on.

These classes are stored in the `app/Mail` directory.

You can create your mailable class using the `make:mail` Artisan command:

```shell
php artisan make:mail OrderProcessing 
```

A Mailable class will contain by default those 3 methods: the `envelope`, `content`, and `attachments` method.

- `envelope`: it returns an `Illuminate\Mail\Mailables\Envelope` object. which is used to define things like the Sender, the Subject, and also sometimes the Recipients.

- `content`: returns an `Illuminate\Mail\Mailables\Content` object that defines the Blade template that will be used to generate the message content.

- `attachments`: returns an array of the email attachments.

**Mailble example:**

```php
class OrderProcessing extends Mailable
{
    public function __construct(protected Customer $customer)
    {
        //
    }

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Order Processing',
        );
    }

    public function content(): Content
    {
        return new Content(
            view: 'mail.order-processing',
        );
    }

    public function attachments(): array
    {
        return [];
    }
}
```

# 3. Email Sender

the **sender** is who the email is going to be "from". There are two ways to configure the sender.

#### a- Using the Envelope:

you may specify the "from" address on your mailable envelope:

```php
public function envelope(): Envelope
{
    return new Envelope(
        from: new Address('jeffrey@example.com', 'Jeffrey Way'),
        subject: 'Order Shipped',
    );
}
```

#### b- Using a Global `from` Address:

You may specify a global "from" address in your `config/mail.php` configuration file. This address will be used if no other "from" address is specified within the mailable class:

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

# 4. Email Content

Within a mailable class's `content` method, you may define the `view` which should be used when rendering the email's contents.

Since each email typically uses a Blade template to render its contents, you have the full power and convenience of the Blade templating engine when building your email's content.

```php
public function content(): Content
{
    return new Content(
        view: 'mail.orders.shipped',
    );
}
```

#### a- Markdown messages:

Laravel provides a pre-built templates and components of email content.

It gives you beautiful, responsive HTML templates for the messages while also automatically generating a plain-text counterpart.

Use the `markdown` parameter instead of the `view` parameter on the `content` method:

```php
public function content(): Content
{
    return new Content(
        markdown: 'mail.orders.shipped',
    );
}
```

Markdown mailables use a combination of Blade components and Markdown syntax which allow you to easily construct mail messages while leveraging Laravel's pre-built email UI components:

```php
<x-mail::message>
# Order Shipped
 
Your order has been shipped!
 
<x-mail::button :url="$url">
View Order
</x-mail::button>
 
Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

You may export all of the Markdown mail components to your own application for customization. To export the components, use the `vendor:publish` Artisan command to publish the `laravel-mail` asset tag:

```shell
php artisan vendor:publish --tag=laravel-mail
```

# 5. Pass Data To View

Typically, you will want to pass some data to your view that you can utilize when rendering the email's HTML.

#### a- Public Properties:

 First, **any <u>public</u> property** defined on your mailable class will automatically be made available to the view.

 For example, you may pass data into your mailable class's constructor and set that data to public properties defined on the class:

```php
class OrderShipped extends Mailable
{
    
    public $foo = 'bar';
    
    public function __construct(
        public Order $order,
    ) {}
 
    public function content(): Content
    {
        return new Content(
            // $foo and $order will be available on that view
            view: 'mail.orders.shipped',
        );
    }
}
```

#### b- with Parameter:

In case you don't want to pass the whole object to the view you can manually pass your data to the view via the `Content` definition's `with` parameter.

You should set this data to `protected` or `private` properties so the data is not automatically made available to the template:

```php
class OrderShipped extends Mailable
{    
    public function __construct(
        private Order $order,
    ) {}
 
    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
            with: [
                'orderName' => $this->order->name,
                'orderPrice' => $this->order->price,
            ],
        );
    }
}
```

# 6. Sending Emails

the **<u><mark>Mail</mark></u>** Facade is used to send emails on laravel. `Mail` has two main methods:

- `to()`: to set the recipient

- `send()`: to set an instance of the **mailable** to send.

The `to` method accepts an email address, a user instance, or a collection of users. If you pass an object or collection of objects, the mailer will automatically use their `email` and `name` properties

```php
public function send(Request $request)
    {
        $order = Order::findOrFail($request->order_id);
 
        Mail::to($request->user())->send(new OrderShipped($order));
 
        return 'Email Sent';
    }
```

By default, Laravel will send email using the mailer configured as the `default` mailer in your application's `mail` configuration file. However, you may use the `mailer` method to send a message using a specific mailer configuration:

```php
Mail::mailer('postmark')
        ->to($request->user())
        ->send(new OrderShipped($order));
```

# 7. Rendering Mailables

When designing a mailable's template, it is convenient to quickly preview the rendered mailable in your browser like a typical Blade template.

For this reason, Laravel allows you to return any mailable directly from a route closure or controller.

When a mailable is returned, it will be rendered and displayed in the browser, allowing you to quickly preview its design without needing to send it to an actual email address:

```php
Route::get('/mailable', function () {
    return new OrderShipped();
});
```

# 8. Events

Laravel dispatches two events while sending mail messages.

The `MessageSending` event is dispatched prior to a message being sent, while the `MessageSent` event is dispatched after a message has been sent.

Remember, these events are dispatched when the mail is being *sent*, not when it is queued. You may create **event listeners** for these events within your application.

# 9. Queueing Mail

Since sending email messages can negatively impact the response time of your application, many developers choose to queue email messages for background sending.

To queue a mail message, use the `queue` method on the `Mail` facade after specifying the message's recipients:

```php
Mail::to($request->user())
    ->queue(new OrderShipped($order));
```

This method will automatically take care of pushing a job onto the queue so the message is sent in the background.

Since this document is not about queues, I suggest you check out this [Laravel Queues documentation](#) to learn more.
