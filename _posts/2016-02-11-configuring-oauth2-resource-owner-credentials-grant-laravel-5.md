---
layout: post
title: Configuring an OAuth2 Resource Owner Password Credentials grant with Laravel 5 and oauth2-server-laravel
description: If you're looking to integrate OAuth2 principles into your Laravel 4/5 application, the oauth2-server-laravel package is a really nice way to do so.
---

If you're looking to integrate OAuth2 principles into your Laravel 4/5 application, the [oauth2-server-laravel](https://github.com/lucadegasperi/oauth2-server-laravel) package is a really nice way to do so. It supports multiple grants out of the box, including:

- Authorization code grant
- Implicit grant
- Resource owner credentials grant
- Client credentials grant
- Refresh token grant

It also allows you to write your own custom grants, should none of the above suit your needs.

I needed a quick solution to get OAuth2 up in an API I'm developing, so I turned to this package to see if it would work for me. Since the API is going to be used by a couple trusted clients (web, mobile), I chose the [Resource Owner Password Credentials grant](http://tools.ietf.org/html/rfc6749#section-4.3). This grant requires the client to authorise with the API by providing it's client ID and secret along with the user credentials.

I decided to note down the steps I took here, as it's the type of configuration I'll likely forget until I come to the next project that requires it.

### Versions

Here is my composer require section for reference:

    "require": {
        "php": ">=5.5.9",
        "laravel/framework": "5.2.*",
        "lucadegasperi/oauth2-server-laravel": "5.1.*"
    },

### Configuration

Assuming the oauth2-server-laravel package has been installed and migrations are up to date (installation docs can be found [here](https://github.com/lucadegasperi/oauth2-server-laravel/blob/master/docs/README.md)), the first step is to configure your application to use the Resource owner credentials grant:

__/config/oauth2.php:__

    'grant_types' => [
        'password' => [
            'class' => '\League\OAuth2\Server\Grant\PasswordGrant',
            'callback' => '\App\OAuth2\Verifier\PasswordGrantVerifier@verify',
            'access_token_ttl' => 3600
        ]
    ],

This is pretty straight forward; we're just configuring a new grant with the name `password`. The `password` key can be changed to whatever you like, and will be need to specified later requesting an access token.

The main thing to be concerned with here is the `callback`. This will be invoked once the client has successfully authenticated. We'll need to create a class for this:

__/app/Oauth2/Verifier/PasswordGrantVerifier.php:__

    <?php

    namespace App\OAuth2\Verifier;

    use Illuminate\Support\Facades\Auth;

    class PasswordGrantVerifier
    {
        public function verify($username, $password)
        {
            if (Auth::once([
                'email'    => $username,
                'password' => $password,
            ])) {
                return Auth::user()->id;
            }
            return false;
        }
    }

This class simply takes a username and password and attempts to find a user in the database. In this case it is making use of Laravel's `Auth` facade to attempt to log the user in for the current request only before returning the user ID on success. This is favourable when building an API as we don't want to persist user sessions between requests - each request should require fresh authentication. You may use any authentication method you choose here, but you'll need to return the user ID (oauth2-server-laravel will use this to associate the user session with an access token).

Next we need to create an endpoint so our client can request an access token:

__/app/Http/routes.php:__

    Route::post('oauth/request', function() {
        return Response::json(Authorizer::issueAccessToken());
    });

Here we're simply creating a `POST` route that fires off oauth2-server-laravel's `Authorizer`. This is where the magic happens - the client will attempt to use the parameters given to authorise the client and authenticate a user. The response will be output as JSON.

Lets set up some resource routes now and protect them with OAuth2:

__/app/Http/routes.php:__

    Route::group(['middleware' => ['oauth']], function () {
        Route::resource('user', 'UserController');
    });

We've used Laravel's group functionality to apply the OAuth middleware to multiple routes in one call. Any request to an endpoing in this group will require a valid access token.

For the sake of completeness, heres the controller:

__/app/Http/Controllers/UserController.php:__

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;
    use App\User;

    class UserController extends Controller
    {
        public function index()
        {
            return User::all();
        }
    }

Now that we've got a resource, let's try and request it without authenticating to check our middleware is working:

    curl http://api.myproject.dev/user

If your middleware is working correctly, you should receive an error explaining that the request is missing an `access_token`:

    {"error":"invalid_request","error_description":"The request is missing a required parameter, includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed. Check the \"access token\" parameter."}

Ok, let's include a dummy token:

    curl http://api.myproject.dev/user\?access_token\=999
    {"error":"access_denied","error_description":"The resource owner or authorization server denied the request."}

It looks like our middleware is protecting unauthorized requests to our resources. So far so good!

### Requesting Resources

Now we'll try and request an access token so we can start accessing our resources. We'll first need to create a client in the `oauth_clients` database table:

    INSERT INTO oauth_clients(id, secret, name, created_at, updated_at) VALUES (1, 12345, 'MyClient', NOW(), NOW());

Of course, you wouldn't use such values in a production environment, but these are fine for testing our configuration.
We'll also need a valid user to authenticate, so create one if you haven't already.

Now that we have a client and a user, we can perform our first request. Let's ask for a token:

    curl --data "username=james&password=secret123&client_id=1&client_secret=12345&grant_type=password" http://api.myproject.dev/oauth/request

Note that `grant_type` must match what you've named your grant in the `config/oauth2.php` `grant_types` section.

You should get a successful response:

    {"access_token":"TeffjqH3yoYPLgELVv0xY9OD8M5CUHQ5H76ModAN","token_type":"Bearer","expires_in":3600}

If not, check your route, client and user credentials and try again. Also make sure you're POSTing to your server. The oauth2-server-laravel package usually throws exceptions with meaningful messages, so it should be quite easy to find out where your problem lies.

We should now be able to request the user listing resource that we tried earlier:

    curl http://api.myproject.dev/user\?access_token\=TeffjqH3yoYPLgELVv0xY9OD8M5CUHQ5H76ModAN
    [{"id":1,"username":"james","created_at":"2016-02-10 21:56:20","updated_at":"2016-02-10 21:56:20"}]

Great! Our access token is working. Again, if you're experiencing problems, check your routes and, specifically, your access token. You might want to look in the following database tables to check everything looks right:

- `oauth_sessions` - the `owner_id` should match your user ID
- `oauth_access_tokens` - the `id` should match the `access_token` you're passing through in your request

### Finishing Up

By now, your application is successfully using OAuth2 with the Resource Owner Credentials grant. One improvement I would recommend is to change the `http_headers_only` configuration option from `false` to `true`. This can be found in `config/oauth2.php`, and will force the `access_token` to be passed in your request headers as opposed to in the query string. Of course this is more secure and won't lead to access tokens being caught up in server access logs etc.

Even better, you could use Laravel's `env` feature:

__config/oauth2.php__

    'http_headers_only' => env('OAUTH2_HTTP_HEADERS_ONLY', true),

You'll need to specify this in your .env file:

__.env__

    OAUTH2_HTTP_HEADERS_ONLY=false

This will allow you to require HTTP headers in certain configurations. Personally I like this to be `false` in development as it means I can quickly test a protected endpoint in my browser without needing to set any headers.

I also made the decision to remove the 
`OAuthExceptionHandlerMiddleware::class` from the `App\Http\Kernel::$middleware` array. The reason for this is that I felt that the format in which it renders errors is little bit too hard-coded, and it didn't fit in with my own custom error format. All exceptions, however, extend `League\OAuth2\Server\Exception\OAuthException` which provides detailed information about the exception, so this was fairly easy to integrate into my existing error handler.

### Conclusion

Theres nothing much to write here, and the post is getting quite long now. Hopefully I've written this in a way where it will be useful to me in the future, as well as anyone else who is looking to implement the Resource Owner Credentials grant in a Laravel application. I'll look into writing about the other grant types as and when I experiment with them.

Thanks for reading!

### Resources

Here are some resources pertaining to this post:

- [oauth2-server-laravel on GitHub](https://github.com/lucadegasperi/oauth2-server-laravel)
- [oauth2-server-laravel Documentation](https://github.com/lucadegasperi/oauth2-server-laravel/tree/master/docs)
- [Resource Owner Password Credentials Grant](http://tools.ietf.org/html/rfc6749#section-4.3)

