# ed-laravel-2fa-easy
Laravel: Easy Two-Factor Authentication with Email and SMS

## Set Up 2FA in Laravel’s Backend
To quickly set up authentication, let’s use Laravel Breeze. Here’s how you install it:

```
composer require laravel/breeze --dev
php artisan breeze:install
```

Next, let’s add storage for our verification code and its expiration time. Add two new fields to the default users migration:

**File**: database/migrations/2014_10_12_000000_create_users_table.php

```
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        // ... other fields
        $table->string('two_factor_code')->nullable();
        $table->dateTime('two_factor_expires_at')->nullable();
    });
}
```

Update the _app/Models/User.php_ to include these new fields:

```
class User extends Authenticatable
{
    protected $fillable = [
        'name', 'email', 'password', 
        'two_factor_code', 'two_factor_expires_at',
    ];
    // ...
}
```

Now, let’s add a function in the User model to generate and save our two-factor code:

**File**: app/Models/User.php

```
public function generateTwoFactorCode(): void
{
    $this->timestamps = false;  // Prevent updating the 'updated_at' column
    $this->two_factor_code = rand(100000, 999999);  // Generate a random code
    $this->two_factor_expires_at = now()->addMinutes(10);  // Set expiration time
    $this->save();
}
```

With this, you can use _$user->generateTwoFactorCode()_ whenever you want to generate a new code.

## Sending the Two-Factor Code via Email
Once a user logs in, we should generate and send them a two-factor code. Let’s achieve this with Laravel’s Notification feature:

To generate the required class, use:

```
php artisan make:notification SendTwoFactorCode
```

One of the perks of using Laravel’s Notifications is that you can decide how you want to send the notification — via email, SMS, etc.

Modify the toMail() function within the SendTwoFactorCode class.

**File**: app/Notifications/SendTwoFactorCode.php

```
class SendTwoFactorCode extends Notification
{
    public function toMail(User $notifiable): MailMessage
    {
        return (new MailMessage)
            ->line("Your two-factor code is {$notifiable->two_factor_code}")
            ->action('Verify Here', route('verify.index'))
            ->line('The code will expire in 10 minutes')
            ->line('If you didn't request this, please ignore.');
    }
}
```

We’ll soon define the route('verify.index'), which will also offer an option to re-send the code.

Note: The $notifiable variable refers to the notification's recipient. In this scenario, it's our logged-in user.

Since we’re using Laravel Breeze, we’ll tweak the login process. In the file app/Http/Controllers/Auth/AuthenticatedSessionController.php, adjust the store() method:

```
public function store(LoginRequest $request)
{
    $request->authenticate();
    $request->session()->regenerate();
    $request->user()->generateTwoFactorCode();
    $request->user()->notify(new SendTwoFactorCode());
    return redirect()->intended(RouteServiceProvider::HOME);
}
```

With this in place, the user will receive an email with the two-factor code upon logging in.

## Setting Up Two-Factor Verification Redirect
To enhance security, it’s vital to ensure users input a verification code after they’ve logged in. Here’s a step-by-step process to achieve this in Laravel:

Initiate a new middleware named TwoFactorMiddleware:

```
php artisan make:middleware TwoFactorMiddleware
```

Modify app/Http/Middleware/TwoFactorMiddleware.php to:

```
class TwoFactorMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $user = auth()->user();
    if (auth()->check() && $user->two_factor_code) {
            if ($user->two_factor_expires_at < now()) {
                $user->resetTwoFactorCode();
                auth()->logout();
                return redirect()->route('login')
                    ->withStatus('Your verification code expired. Please re-login.');
            }
            if (!$request->is('verify*')) {
                return redirect()->route('verify.index');
            }
        }
        return $next($request);
    }
}
```

This middleware checks if a user has a set two-factor code. If it’s expired, it resets the code and prompts the user to log in again. If not, the user is redirected to a verification form.

Update app/Models/User.php with a function to reset the two-factor code:

```
public function resetTwoFactorCode(): void
{
    $this->timestamps = false;
    $this->two_factor_code = null;
    $this->two_factor_expires_at = null;
    $this->save();
}
```

Now, list the middleware in app/Http/Kernel.php:

```
class Kernel extends HttpKernel
{
    protected $routeMiddleware = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        'twofactor' => \App\Http\Middleware\TwoFactorMiddleware::class,
    ];
}
```

Secure your dashboard by updating routes/web.php:

```
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'twofactor'])->name('dashboard');
```

Now, any attempt to access the dashboard will be secured with two-factor authentication.


## Building the Code Verification Page in Laravel

Let’s set up the page to input the verification code and handle its processing in Laravel.

First, create a new controller named TwoFactorController:

```
php artisan make:controller TwoFactorController
```

In routes/web.php, add these routes for the controller:

```
Route::middleware(['auth', 'twofactor'])->group(function () {
    Route::get('verify/resend', [TwoFactorController::class, 'resend'])->name('verify.resend');
    Route::resource('verify', TwoFactorController::class)->only(['index', 'store']);
});
```

Inside app/Http/Controllers/TwoFactorController, the logic unfolds as:

```
class TwoFactorController extends Controller
{
    public function index(): View
    {
        return view('auth.twoFactor');
    }
public function store(Request $request): ValidationException|RedirectResponse
    {
        $request->validate([
            'two_factor_code' => ['integer', 'required'],
        ]);
        $user = auth()->user();
        if ($request->input('two_factor_code') !== $user->two_factor_code) {
            throw ValidationException::withMessages([
                'two_factor_code' => __('The code you entered doesn't match our records'),
            ]);
        }
        $user->resetTwoFactorCode();
        return redirect()->to(RouteServiceProvider::HOME);
    }
    public function resend(): RedirectResponse
    {
        $user = auth()->user();
        $user->generateTwoFactorCode();
        $user->notify(new SendTwoFactorCode());
        return redirect()->back()->withStatus(__('Code has been sent again'));
    }
}
```

This controller handles three primary functions:

- Displaying the verification form (index method).
- Verifying the input code (store method).
- Resending the code if needed (resend method).

Your form can be placed in resources/views/auth/twoFactor.blade.php. It can be designed using the built-in components from Laravel Breeze:

```
<x-guest-layout>
    <x-auth-card>
        <x-slot name="logo">
            <a href="/">
                <x-application-logo class="h-20 w-20 fill-current text-gray-500" />
            </a>
        </x-slot>
        <div class="mb-4 text-sm text-gray-600">
            {{ new Illuminate\Support\HtmlString(__("Received an email with a login code? If not, click <a class=\"hover:underline\" href=\":url\">here</a>.", ['url' => route('verify.resend')])) }}
        </div>
        <x-auth-session-status class="mb-4" :status="session('status')" />
        <form method="POST" action="{{ route('verify.store') }}">
            @csrf
            <div>
                <x-input-label for="two_factor_code" :value="__('Code')" />
                <x-text-input id="two_factor_code" class="mt-1 block w-full" type="text" name="two_factor_code" required autofocus />
                <x-input-error :messages="$errors->get('two_factor_code')" class="mt-2" />
            </div>
            <div class="mt-4 flex justify-end">
                <x-primary-button>
                    {{ __('Submit') }}
                </x-primary-button>
            </div>
        </form>
    </x-auth-card>
</x-guest-layout>
```

With this in place, you’ve successfully set up two-factor authentication via email in Laravel. If you’re considering using SMS, let’s delve into that next.

## Setting Up SMS Verification in Laravel

Want to add another layer of security by sending verification codes through SMS? Let’s dive in.

Want to add another layer of security by sending verification codes through SMS? Let’s dive in.

First, we’ll configure the notification channels. In **config/auth.php**, introduce these settings:

```
'two_factor' => [
    'via' => ['mail'],
],
```

The 'via' key can either be 'mail' or 'vonage'. We'll use Vonage (previously Nexmo) for SMS.

Run the following command:

```
composer require laravel/vonage-notification-channel
```

Ensure you configure the Vonage environment variables properly.

```
class SendTwoFactorCode extends Notification
{
    public function via($notifiable): array
    {
        return config('auth.two_factor.via');
    }
    public function toMail(User $notifiable): MailMessage
    {
        return (new MailMessage)
            ->line("Your verification code: {$notifiable->two_factor_code}")
            ->action('Confirm Here', route('verify.index'))
            ->line('This code expires in 10 minutes')
            ->line('Didn't request this code? Just ignore.');
    }
    public function toVonage($notifiable): VonageMessage
    {
        return (new VonageMessage())
            ->content("Your verification code: {$notifiable->two_factor_code}");
    }
}
```

This class now fetches the delivery method from the config and also defines a new method for Vonage.

We need to store users’ phone numbers. Thus, make a modification to the users table:

```
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        // ... Other columns
        $table->string('phone_number');
        // ... Rest of the columns
    });
}
```

Don’t forget to include the phone number in the User model's $fillable property and provide a method for Vonage:

```
class User extends Authenticatable
{
    protected $fillable = [
        // ... Other attributes
        'phone_number',
        'two_factor_code',
        'two_factor_expires_at',
    ];
    public function routeNotificationForVonage($notification)
    {
        return $this->phone_number;
    }
}
```

Add a phone number input field in resources/views/auth/register.blade.php.

```
<!-- Phone Number -->
<div class="mt-4">
    <x-input-label for="phone_number" :value="__('Phone Number')" />
    <x-text-input id="phone_number" class="block mt-1 w-full" type="text" name="phone_number" :value="old('phone_number')" required />
    <x-input-error :messages="$errors->get('phone_number')" class="mt-2" />
</div>
```

Update the store method of app/Http/Controllers/Auth/RegisteredUserController.php to handle the phone number:

```
public function store(Request $request)
{
    $request->validate([
        // ... Other validations
        'phone_number' => ['required'],
    ]);
$user = User::create([
        // ... Other fields
        'phone_number' => $request->phone_number,
    ]);
    event(new Registered($user));
    Auth::login($user);
    return redirect(RouteServiceProvider::HOME);
}
```

With these steps, you’ve enhanced your Laravel app’s security using SMS for verification.

