# Google Two-Factor Authentication (2FA) Integration in Laravel

This guide explains how to integrate Google Two-Factor Authentication (2FA) in a Laravel project. The project uses the [`pragmarx/google2fa-laravel`](https://github.com/antonioribeiro/google2fa-laravel) package to secure user authentication with Google Authenticator.

## Requirements

- PHP >= 7.3
- Laravel >= 8.x
- Composer

## Steps to Integrate Google 2FA

### 1. Install the Package

First, you need to install the `pragmarx/google2fa-laravel` package using Composer:

```bash
composer require pragmarx/google2fa-laravel
```
### 2. Publish the Configuration File

Next, publish the configuration file for Google 2FA:

```bash
php artisan vendor:publish --provider="PragmaRX\Google2FALaravel\ServiceProvider"

```

### 3. Add the Google 2FA Secret Column to the Users Table

You need to add a column to your users table to store the 2FA secret key. Run the following Artisan command to create a migration:

```bash
php artisan make:migration add_google2fa_secret_to_users_table --table=users

```
In the generated migration file, add the column:

```bash
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('google2fa_secret')->nullable();
    });
}

public function down()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('google2fa_secret');
    });
}


```
Run the migration:
```bash
php artisan migrate

```
### 4. Update the User Model

In the User model, add the google2fa_secret field to the $fillable array:


```bash
protected $fillable = [
    'name', 'email', 'password', 'google2fa_secret'
];

```
### 5. Generate and Store the 2FA Secret Key

In your controller (e.g., UserController), add the following method to generate and store the 2FA secret key for the user:

```bash
use Google2FA;

public function enable2fa(Request $request)
{
    $user = $request->user();

    // Generate a secret key
    $google2fa = app('pragmarx.google2fa');
    $secretKey = $google2fa->generateSecretKey();

    // Store the secret key in the user's table
    $user->google2fa_secret = $secretKey;
    $user->save();

    // Generate the QR code for Google Authenticator
    $QR_Image = $google2fa->getQRCodeInline(
        config('app.name'),
        $user->email,
        $secretKey
    );

    return view('2fa.enable', ['QR_Image' => $QR_Image, 'secret' => $secretKey]);
}

```
### 6. Create the Enable 2FA View

Create a new Blade view resources/views/2fa/enable.blade.php to display the QR code for Google Authenticator:

```bash
<h2>Enable Two-Factor Authentication</h2>
<p>Scan the following QR code with your Google Authenticator app:</p>
<img src="{{ $QR_Image }}" alt="QR Code">
<p>Alternatively, enter this secret key manually: {{ $secret }}</p>

```
### 7. Verify 2FA Code

Add a method in your controller to verify the 2FA code provided by the user:

```bash
public function verify2fa(Request $request)
{
    $user = $request->user();

    $google2fa = app('pragmarx.google2fa');

    // Verify the OTP entered by the user
    $valid = $google2fa->verifyKey($user->google2fa_secret, $request->input('2fa_code'));

    if ($valid) {
        // 2FA verification passed, allow access
        return redirect()->intended('dashboard');
    } else {
        // Verification failed
        return redirect()->back()->withErrors(['2fa_code' => 'The 2FA code is incorrect.']);
    }
}

```
### 8. Add a Verification Form

Create a Blade view for the 2FA verification form (resources/views/2fa/verify.blade.php):

```bash
<form method="POST" action="{{ route('2fa.verify') }}">
    @csrf
    <label for="2fa_code">Two-Factor Authentication Code</label>
    <input id="2fa_code" type="text" name="2fa_code" required>
    <button type="submit">Verify</button>
</form>

```
### 9. Protect Routes with 2FA Middleware

Create middleware to ensure that users complete 2FA verification before accessing certain routes. 
Run:

```bash
php artisan make:middleware TwoFactorMiddleware

```
Modify the middleware logic:
```bash
public function handle($request, Closure $next)
{
    $user = $request->user();

    if ($user && $user->google2fa_secret && !$request->session()->has('2fa_verified')) {
        return redirect()->route('2fa.verify');
    }

    return $next($request);
}

```
Register the middleware in app/Http/Kernel.php:
```bash
protected $routeMiddleware = [
    '2fa' => \App\Http\Middleware\TwoFactorMiddleware::class,
];

```
### 10. Protect Routes with 2FA
Apply the 2fa middleware to the routes you want to protect:

```bash
Route::group(['middleware' => ['auth', '2fa']], function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    });
});

```
### 11. Redirect to 2FA Verification after Login
Modify the login controller (AuthenticatedSessionController.php) to redirect users with 2FA enabled to the verification page:

```bash
protected function authenticated(Request $request, $user)
{
    if ($user->google2fa_secret) {
        return redirect()->route('2fa.verify');
    }

    return redirect()->intended('dashboard');
}


```
# That's it! You have successfully integrated Google 2FA into your Laravel project.
