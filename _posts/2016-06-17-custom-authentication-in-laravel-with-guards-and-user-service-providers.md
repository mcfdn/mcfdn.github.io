---
layout: post
title: Custom Authentication in Laravel with Guards and User Service Providers
description: Laravel provides a quick and easy way to enable user authentication out of the box through it's AuthManager and guards.
---

Laravel provides a quick and easy way to enable user authentication out of the box through it's AuthManager, EloquentUserProvider and guards. While this is great, sometimes extra flexibility is needed. An example might be when building a separate login for users with administrative rights, where those users are stored in the same table as regular system users. In this situation you might want to authenticate based on `username`, `password` and another attribute such as `is_admin`.

The [documentation](https://laravel.com/docs/5.2/authentication#authenticating-users) highlights that it's possible to manually authenticate users, but that doesn't work for those of us who want to take advantage of the really helpful `AuthenticatesAndRegistersUsers` trait that does a lot of the controller work for you.

### Software Versions

- laravel/framework 5.2.*

### The Solution

So, lets assume we're building an admin interface for an existing system, where the administrators are also regular users. Our migration looks like this:

    <?php

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Migrations\Migration;

    class CreateUsersTable extends Migration
    {
        public function up()
        {
            Schema::create('users', function (Blueprint $table) {
                $table->increments('id');
                $table->string('email')->unique();
                $table->string('password', 60);
                $table->tinyInteger('is_admin')->default(0);
                $table->rememberToken();
                $table->timestamps();

                $table->index('email');
                $table->index('is_admin');
            });
        }

        public function down()
        {
            Schema::drop('users');
        }
    }

It's basically the same as the default `CreateUsersTable` migration, but we've added `is_admin` and a couple of indicies.

We can go ahead and create a user. Assuming you have the model set up:

    $user = new \App\User();
    $user->email = 'user@app.com';
    $user->password = \Hash::make('secret');
    $user->is_admin = 0;

    $admin = new \App\User();
    $admin->email = 'admin@app.com';
    $admin->password = \Hash::make('secret');
    $admin->is_admin = 1;

    $user->save();
    $admin->save();

Now we have a couple of test users set up so we can play around with our authentication.

### Auth Configuration

Authentication in Laravel is centered around [guards](https://laravel.com/docs/5.2/authentication) which allow us to specify varying ways that a user can be authenticated. The first thing we'll do is add our own guard for authenticating admin users. We can do this in `/config/auth.php`:

<strong class="code-title">/config/auth.php</strong>

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'token',
            'provider' => 'users',
        ],

        'admin' => [
            'driver' => 'session',
            'provider' => 'admins'
        ],
    ],

Here we've added a new guard named `admin`. We'll be using sessions as the driver as well as a custom `admins` guard provider. We'll configure the provider in the `providers` section of the same configuration:

<strong class="code-title">/config/auth.php</strong>

    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\User::class,
        ],

        'admins' => [
            'driver' => 'eloquent.admin',
            'model' => App\User::class,
        ],
    ],

### The User Provider

Now that we've specified a custom provider, we need to build it. The guard provider allows the retrieval of users from your storage engine, and follows the `Illuminate\Contracts\Auth\UserProvider` contract. Since we're using Eloquent in this example, we can extend Laravel's default `EloquentUserProvider` with our own, providing new functionality where required:

<strong class="code-title">/app/Auth/EloquentAdminUserProvider.php</strong>

    <?php

    namespace App\Auth;

    use Illuminate\Auth\EloquentUserProvider;
    use Illuminate\Support\Str;

    class EloquentAdminUserProvider extends EloquentUserProvider
    {
        public function retrieveByCredentials(array $credentials)
        {
            // Of course here, you could perform the query yourself with the is_admin comparison, but
            // I think it's best to avoid as much duplication as possible
            $user = parent::retrieveByCredentials($credentials);

            return $user->is_admin === false
                    ? null
                    : $user;
        }
    }

The only method we need to override is `retrieveByCredentials`. Above, I've chosen only to override some of it, so all we need to do is check if any user that is found with the specified username and password in the parent class is also an admin.

In our auth configuration earlier, we referenced our new guard provider as `eloquent.admin`. We'll need to bind that reference to our implementation, which we can accomplish in the `AuthServiceProvider`:

<strong class="code-title">/app/Providers/AuthServiceProvider.php</strong>

    <?php

    namespace App\Providers;

    use Auth;
    use Illuminate\Contracts\Auth\Access\Gate as GateContract;
    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
    use App\Auth\EloquentAdminUserProvider;
    use App\User;

    class AuthServiceProvider extends ServiceProvider
    {
        protected $policies = [
            'App\Model' => 'App\Policies\ModelPolicy',
        ];

        public function boot(GateContract $gate)
        {
            $this->registerPolicies($gate);

            // Binding eloquent.admin to our EloquentAdminUserProvider
            Auth::provider('eloquent.admin', function($app, array $config) {
                return new EloquentAdminUserProvider($app['hash'], $config['model']);
            });
        }
    }

We also pass in the app hash, which is used to verify a user's password, as well as the name of the model representing the user record.

### Middleware

Next, we'll set up our own admin authentication middleware. This will be similar to the default `Authenticate` middleware, but will also check that the user is an admin, and will be used on all admin routes:

<strong class="code-title">/app/Http/Middleware/AuthenticateAdmin.php</strong>

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Support\Facades\Auth;

    class AuthenticateAdmin
    {
        public function handle($request, Closure $next, $guard = null)
        {
            if (Auth::guard($guard)->guest() || !Auth::guard($guard)->user()->is_admin) {
                if ($request->ajax() || $request->wantsJson()) {
                    return response('Unauthorized.', 401);
                } else {
                    return abort(401);
                }
            }
            return $next($request);
        }
    }

Here, if the user is a guest or not an admin, we return a 401 response, otherwise we continue on with the rest of the middlware stack. Note that we cannot use `Auth::user()`; we must first explicitly pass in the name of the guard we are using (admin).

Don't forget to register your middleware in the kernel:

<strong class="code-title">/app/Http/Kernel.php</strong>

    protected $routeMiddleware = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        'auth.admin' => \App\Http\Middleware\AuthenticateAdmin::class,
        'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    ];

### Routing

Now we can use our middleware and guard while writing our routes:

<strong class="code-title">/app/Http/routes.php</strong>

    Route::group(['prefix' => 'admin'], function() {
        Route::get('/login', ['as' => 'admin.getLogin', 'uses' => 'Admin\AuthController@getLogin']);
        Route::post('/login', ['as' => 'admin.postLogin', 'uses' => 'Admin\AuthController@postLogin']);
        Route::get('/logout', ['as' => 'admin.getLogout', 'uses' => 'Admin\AuthController@getLogout']);

        Route::group(['middleware' => ['auth.admin:admin']], function() {
            Route::get('/', ['as' => 'admin.home', 'uses' => 'Admin\IndexController@getIndex']);
        });
    });

We've set up a route group with the prefix of admin. Our `admin.home` route is protected by our `AuthenticateAdmin` middleware as `auth.admin`, as well as our `admin` guard as specified in our auth config. Note that you must use `getLogin`, `postLogin` and `getLogout` as your action names since we'll be using the `AuthenticatesUsers` trait which defines them as so.

We now have the following routes available:

    GET /admin/login   <-- open
    POST /admin/login  <-- open
    GET /admin/logout  <-- open
    GET /admin         <-- guarded; authentication required

### Controllers

We're almost there! Now we need to set up our controller. Here we begin to see how easy providing authentication is, as the actions are set up for us already in `AuthenticatesAndRegistersUsers`:

<strong class="code-title">/app/Http/Controllers/Admin/AuthController.php</strong>

    <?php

    namespace App\Http\Controllers\Admin;

    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;
    use Illuminate\Foundation\Auth\AuthenticatesAndRegistersUsers;

    class AuthController extends Controller
    {
        use AuthenticatesAndRegistersUsers;

        protected $loginView = 'admin.auth.login';

        protected $guard = 'admin';

        protected $redirectTo = null;

        public function __construct()
        {
            $this->redirectTo = route('admin.home');
        }
    }

With minimal configuration, we've specified the view we want to use for our login, the guard we wish to use when authenticating and finally the URI that should be redirected to on success. I prefer to use routes so I set this in the constructor.

We'll quickly set up the controller to handle the `admin.home` route:

<strong class="code-title">/app/Http/Controllers/Admin/IndexController.php</strong>

    <?php

    namespace App\Http\Controllers\Admin;

    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class IndexController extends Controller
    {
        public function getIndex()
        {
            return 'AUTHENTICATED IN ' . __METHOD__;
        }
    }

Finally, lets create a simple login view so we can actually authenticate ourselves:

<strong class="code-title">/resources/views/admin/auth/login.blade.php</strong>

    {!! \Form::open(['route' => 'admin.postLogin']) !!}

    {!! \Form::text('email') !!}
    {!! \Form::password('password') !!}
    {!! \Form::submit() !!}

    {!! \Form::close() !!}

Note that I'm using [Laravel Collective](https://laravelcollective.com/docs/5.2/html#opening-a-form) here, but of course you can use raw HTML if you prefer.

### Testing

We're now ready to authenticate ourselves. Let's check our middleware is working by visiting the `admin.home` route. You should receive a 401 error as you're not yet logged in.

Now, we'll try authenticating with a user without administrative rights. Visit the `admin.getLogin` route and attempt to login with a user without administrative powers; `user@app.com:secret`. You should be redirected back to the `admin.getLogin` route as you're not an administrator!

Finally, we'll try our admin; `admin@app.com:secret`. If everything is set up correctly, you should be redirected to the `admin.home` route and see the following:

    AUTHENTICATED IN App\Http\Controllers\Admin\IndexController::getIndex

This proves both your middleware and `EloquentAdminUserProvider` are working as intended.

You can now logout using the `admin.getLogout` URI.

That's all there is to it. It feels like quite a bit to set up, but what you're left with is the flexibility to choose how your users are retrieved for authentication purposes, as well as making use of the great behind the scenes functionality that Laravel provides for authentication.

### Resources

- [Laravel Authentication Documentation](https://laravel.com/docs/5.2/authentication)
- [Laravel Collective](https://laravelcollective.com/docs/5.2/html)







