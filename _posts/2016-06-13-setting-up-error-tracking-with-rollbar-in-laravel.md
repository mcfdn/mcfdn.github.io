---
layout: post
title: Setting up Error Tracking with Rollbar in Laravel
description: Error tracking in applications is often overlooked. Sometimes it's difficult to know exactly how to collate and organise your errors until it's too late.
---

Error tracking in applications is often overlooked. Sometimes it's difficult to know exactly how to collate and organise your errors until it's too late. There are many solutions out there that allow you to offload this burden, one of which is [Rollbar](https://rollbar.com).

Rollbar is a free error tracking solution that provides libraries for many languages. Here's how I integrated their PHP library into a Laravel 5.2 project of mine.

### Software Versions

These are the software versions I used in this post:

- laravel/framework 5.2.*
- rollbar/rollbar-php ~0.18.0

If you're using something different, the steps outlined here may not work for you.

### Configuration

After setting up your project in your Rollbar account, the first step is to install the Rollbar PHP library. You'll probably have already done this while setting up your project, but if you haven't, you can follow the instructions on the [GitHub page](https://github.com/rollbar/rollbar-php).

Next, we'll configure Rollbar. Append the following configuration to your application config array in `/config/app.php`:

<strong class="code-title">/config/app.php</strong>

    'rollbar' => [
        'access_token' => env('ROLLBAR_ACCESS_TOKEN'),
        'environment' => env('APP_ENV', 'production'),
        'root' => base_path(),
    ],

These rely on environment configuration values, so go be sure to add them to your `.env` file:

<strong class="code-title">/.env</strong>

    ROLLBAR_ACCESS_TOKEN=YOUR_ACCESS_TOKEN_GOES_HERE
    APP_ENV=local

You probably already have an `APP_ENV` entry in here, so only add it if you need to.

### Middleware

We'll set up Rollbar as part of your application middleware so it's available in every request. Lets create a new middleware file:

<strong class="code-title">/app/Http/Middleware/Rollbar.php</strong>

    <?php

    namespace App\Http\Middleware;

    use Closure;

    class Rollbar
    {
        /**
         - @param  \Illuminate\Http\Request  $request
         - @param  \Closure  $next
         - @param  string|null  $guard
         - @return mixed
         */
        public function handle($request, Closure $next, $guard = null)
        {
            \Rollbar::init(config('app.rollbar'), false, false);

            return $next($request);
        }
    }

This is a simple class which sets up Rollbar with your configuration. I chose to prevent Rollbar from setting up it's own exception and error handler with the second and third arguments here, but this is your choice.

Now we just need to register our middleware in the kernel:

<strong class="code-title">/app/Http/Kernel.php</strong>

    protected $middleware = [
        \App\Http\Middleware\Rollbar::class,
        \Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode::class,
    ];

With that, Rollbar is now set up and ready to use!

### Exception Handler

A good place to begin using Rollbar is in your exception handler:

<strong class="code-title">/app/Exceptions/Handler.php</strong>

    /**
     * Report or log an exception.
     *
     * This is a great spot to send exceptions to Sentry, Bugsnag, etc.
     *
     * @param  \Exception  $e
     * @return void
     */
    public function report(Exception $e)
    {
        \Rollbar::report_exception($e);

        parent::report($e);
    }


This is part of the default exception handler provided with Laravel, but we've just ensured that any exceptions are reported to Rollbar before following the default process.

You can test everything is working by forcing an error in any route/controller:

    abort(500, 'Testing out Rollbar');

Your Rollbar dashboard should show your error!

That's pretty much it. By following this, you'll have all your exceptions being reported to Rollbar, but you shouldn't stop there. Rollbar allows you to report other errors and generic messages too, so use it to make your applications more robust.

### Resources

- [Rollbar](https://rollbar.com)
- [HTTP Middleware](https://laravel.com/docs/5.2/middleware)
- [The Exception Handler](https://laravel.com/docs/5.2/errors#the-exception-handler)
- [Rollbar PHP on GitHub](https://github.com/rollbar/rollbar-php)

