---
layout: post
title: Customising API Responses in Laravel 5
description: Laravel makes it really easy to return a JSON response from the API you're building; you simply return the value from your route or controller action and it will be JSON encoded automatically, ready to be consumed.
---

This post uses **Laravel Version: 5.2.41**.

Laravel makes returning JSON responses from the API you're building trivial; you simply return the value from your route or controller action and it will be JSON encoded automatically, ready to be consumed.

The framework also makes it easy to hide, show and alter your model data through the `$hidden` and `$appends` attributes:

    <?php

    namespace App;

    class User extends Model
    {
        protected $hidden = ['admin'];

        protected $appends = ['is_admin'];

        public function getIsAdminAttribute()
        {
            return $this->attributes['is_admin'] = (bool) $this->attributes['admin'];
        }
    }

Paginated results are also formatted nicely; merely return a `LengthAwarePaginator` instance in your route or controller and your response will contain collection meta data such as count, page number and next and previous URLs:

    {
        "total": 200,
        "per_page": 50,
        "current_page": 1,
        "last_page": 4,
        "next_page_url": "http://my-api.dev/api/v1/my-resource?page=2",
        "prev_page_url": null,
        "from": 1,
        "to": 50,
        "data": [
            ...
        ]
    }

However, sometimes you'll want a more custom response for your API, or your web application in general; maybe you want to envelope your responses or include some extra data about the request or response. We'll explore a couple of ways you can do so; **response macros** and **middleware**.

## Response Macros

Laravel supports the registration of custom macros for responses. An example illustrates how a macro can be registered in a service provider:

<strong class="code-title">/app/Providers/ResponseServiceProvider.php</strong>

    <?php

    namespace App\Providers;

    use Illuminate\Support\ServiceProvider;
    use Illuminate\Contracts\Routing\ResponseFactory;

    class ResponseServiceProvider extends ServiceProvider
    {
        public function boot(ResponseFactory $factory)
        {
            $factory->macro('api', function ($data) use ($factory) {
                
                $customFormat = [
                    'success' => true,
                    'data' => $data
                ];
                return $factory->make($customFormat);
            });
        }

        public function register()
        {
            //
        }
    }

Here we're registering a custom `api` macro, which wraps the original response data in a new array before creating a new response object for it. We'll need to register our service provider in the `providers` section of our application config:

<strong class="code-title">/config/app.php</strong>

    return [

        // ...

        'providers' => [
            // ...
            App\Providers\ResponseServiceProvider::class,
        ],

        // ...
    ];

With that done, we can use the response macro in our routes and controllers like so:

    Route::get('/user', function () {
        $users = User::all();

        return response()->api($users);
    });

The `/user` route will now return the following JSON:

    {
        "success": true,
        "data" : [
            ...
        ]
    }

This is a simple and [recommended](https://laravel.com/docs/5.2/responses#response-macros) way of customising Laravel responses.

## Middleware

So what if you don't want to have to make a call to `response()->myMacro()`? Perhaps you have many controller actions that you don't want to have to update, or maybe you want to ensure every response is forced into a particular format by default.

The Laravel [middleware](https://laravel.com/docs/5.2/middleware) mechanism provides a simple way to achieve this:

<strong class="code-title">/app/Http/Middleware/BuildHttpResponse.php</strong>

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Response;

    class BuildApiResponse
    {
        /**
         - Handle an incoming request.
         *
         - @param  \Illuminate\Http\Request  $request
         - @param  \Closure  $next
         - @param  string|null  $guard
         - @return mixed
         */
        public function handle($request, Closure $next, $guard = null)
        {
            $response = $next($request);
            $original = $response->getOriginalContent();

            $response->setContent([
                'success' => true,
                'data' => $original
            ]);
            return $response;
        }
    }

Here we wait for the remaining middleware to finish executing before we alter the response to our own specification.

We'll need to register our middleware in the HTTP kernel:

<strong class="code-title">/app/Http/Kernel.php</strong>

    protected $middlewareGroups = [
        'web' => [
            // ...
        ],

        'api' => [
            // ...
            \App\Http\Middleware\BuildApiResponse::class,
        ],
    ];

So there you have it - two simple ways to customise the response format that a Laravel application provides. While the majority of use cases for this level of configuration will be API focused, as is this post, I'm sure there are many situations where customisation will be necessary for other response types. These examples should be suitable for most requirements.

Thanks!
