![](laravel%20auth.png)

# Table of Contents

1. [Introduction](#1-introduction)
2. [Custom Authentication (Without Starter Kits)](#2-custom-authentication-without-starter-kits)
   - [Key Auth Methods](#a-key-auth-methods)
   - [Session Management](#b-session-management)
   - [Login Example](#c-login-example)
   - [Logout Example](#d-logout-example)
   - [Protecting Routess](#e-protecting-routes)
   - [Email Verification](#f-email-verification)
   - [Password Reset](#g-password-reset)
   - [Rate Limiting & Throttling](#h-rate-limiting--throttling)
   - [HTTP Basic Authentication](#i-http-basic-authentication)
3. [Authentication with Starter Kits](#3-authentication-with-starter-kits)
   - [Breeze](#a-breeze)
   - [Jetstream](#b-jetstream)
4. [Sanctum](#4-sanctum)
   - [Installation](#a-installation)
   - [Token-Based Authentication](#b-token-based-authentication)
   - [Session-Based Authentication for SPAs](#c-session-based-authentication-for-spas)
5. [Socialite](#5-socialite)
   - [What is OAuth?](#a-what-is-oauth)
   - [Installation](#b-installation)
   - [Configuration](#c-configuration)

# 1. Introduction

Laravel provides multiple ways to implement authentication:

1. **Starter Kits** (e.g., Breeze, Jetstream): Quickly set up auth with pre-built components.
2. **Custom Auth**: Manually use Laravel's `Auth` facade to log users in, out, and verify authentication status.

By default, Laravel uses **session-based authentication**, where user credentials are stored server-side, and session identifiers are stored in cookies. For token-based authentication, **Sanctum** is a common choice, supporting both tokens and sessions (used by starter kits).

# 2. Custom Authentication (Without Starter Kits)

Laravel's `Auth` facade provides essential methods to handle authentication.

#### a. Key `Auth` Methods

- **`Auth::attempt($credentials, $remember = false)`**  Attempts to log in a user. Returns `true` on success, `false` otherwise.
  
  ```php
  $credentials = ['email' => 'example@mail.com', 'password' => 'password'];
  Auth::attempt($credentials, true); // "Remember Me" enabled`
  ```

- **`Auth::check()`**  Returns `true` if a user is authenticated.

- **`Auth::user()` / `Auth::id()`**  Retrieves the authenticated user or their ID. Returns `null` if unauthenticated.

- **`Auth::login($user)`**  Logs in a user instance directly (e.g., after registration).

- **`Auth::logout()`**  Logs out the authenticated user.

## b. Session Management

Use sessions to handle user state:

- **Regenerate Session ID** (after login):
  
  ```php
  $request->session()->regenerate();
  ```

- **Invalidate Session** (on logout):
  
  ```php
  $request->session()->invalidate();
  ```

## c. Login example:

```php
class LoginController extends Controller
{
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required'
        ]);

        if (Auth::attempt($credentials)) {
            $request->session()->regenerate(); // Secure session
            return redirect()->intended('/dashboard'); // Redirect or fallback
        }

        return back()->withErrors([
            'email' => 'Invalid credentials.',
        ]);
    }
}
```

**Note:** `redirect()->intended()` sends users to their original destination if they were redirected to login by middleware. Otherwise, it uses the fallback URL.

## d. Logout Example

```php
class LogoutController extends Controller
{
    public function logout(Request $request)
    {
        Auth::logout(); // Log out the user
        $request->session()->invalidate(); // Clear session
        $request->session()->regenerateToken(); // Prevent CSRF issues

        return redirect('/'); // Redirect to home
    }
}
```

## e. Protecting Routes

Use the built-in `auth` middleware to restrict access to routes for authenticated users:

```php
Route::get('/dashboard', [DashboardController::class, 'index'])->middleware('auth');
```

Update the `redirectTo()` method in `app/Http/Middleware/Authenticate.php` to customize the redirect for unauthenticated users.

## f. Email Verification

Laravel makes it easy to implement email verification to ensure users verify their email addresses before accessing certain parts of the app.

1. **Enabling Email Verification:**
   
   - Add `Illuminate\Contracts\Auth\MustVerifyEmail` to your `User` model:
     
     ```php
     use Illuminate\Contracts\Auth\MustVerifyEmail;
     class User extends Authenticatable implements MustVerifyEmail
     ```
   
   - Ensure the `email_verified_at` column exists in your `users` table migration:
     
     ```php
     $table->timestamp('email_verified_at')->nullable();
     ```
   
   - Protect routes using the `verified` middleware:
     
     ```php
     Route::get('/dashboard', fn() => view('dashboard'))->middleware(['auth', 'verified']);
     ```

2. **Routes for Email Verification:**
   
   Laravel provides built-in routes for:
   
   1. **Verify Email**: `/email/verify` (GET) - Display verification notice.
   2. **Verification Callback**: `/email/verify/{id}/{hash}` (GET) - Handle verification link.
   3. **Resend Verification Email**: `/email/verification-notification` (POST).
   
   You can register these routes using:
   
   ```php
   Auth::routes(['verify' => true]);
   ```

## g. Password Reset

Laravel provides a ready-to-use password reset mechanism, including controllers, views, and notifications.

1. **Setup**
- Ensure `password_resets` table migration exists:
  
  ```php
  $table->string('email')->index();
  $table->string('token');
  $table->timestamp('created_at')->nullable();
  ```

- Use built-in routes for reset functionality:
  
  ```php
  Auth::routes(['reset' => true]);
  ```
2. **Flow**
- **Forgot Password Form**: Users enter their email address to receive a reset link.
- **Reset Link**: A tokenized link is sent via email.
- **Reset Password Form**: Users set a new password.

## h. Rate Limiting & Throttling

Laravel provides built-in rate limiting for login attempts to protect against brute force attacks.

1. **Basic Login Throttling**:
   
   ```php
   // In LoginController or auth route
   public function login(Request $request)
   { 
       $request->validate([
            'email' => 'required|email',
            'password' => 'required',]);
   
       // Limit login attempts
        if (RateLimiter::tooManyAttempts($request->email, 5)) {
            $seconds = RateLimiter::availableIn($request->email);
            return back()->withErrors([
            'email' => "Too many login attempts. Please try again in {$seconds} seconds.",]);
           }
   
       if (Auth::attempt($credentials)) {
            RateLimiter::clear($request->email);
            return redirect()->intended('dashboard');
       }
   
       RateLimiter::hit($request->email);
       return back()->withErrors(['email' => 'Invalid credentials']);
   }
   ```

2. **Using Middleware**:
   
   ```php
   // In routes/web.php
   Route::post('/login', [LoginController::class, 'login'])
       ->middleware(['throttle:login']);
   
   // In app/Http/Kernel.php
   protected $middlewareAliases = [
       'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
   ];
   ```

## i. HTTP Basic Authentication

HTTP Basic Authentication prompts users for credentials via a browser dialog. Laravel simplifies this with the `auth.basic` middleware:

```php
Route::get('/admin', [AdminController::class, 'index'])->middleware('auth.basic');
```

When accessed, the server returns a `401 Unauthorized` response if the client isn't authenticated.

# 3. Authentication with Starter Kits

Laravel Starter Kits provide pre-built routes, controllers, and views to quickly set up authentication.

## a. Breeze:

Laravel **Breeze** is a lightweight and popular starter kit for authentication. It includes features like:

- Registration, Login, and Logout
- Password Reset and Email Verification
- Profile Management (Update name, email, and password)
1. Install Breeze:

```shell
composer require laravel/breeze --dev
```

2. Publish resources:

```shell
php artisan breeze:install
```

    This generates authentication routes, controllers, views, and other resources, giving     you full control over customization.

3. Choose a front-end stack during installation:
   
   - **Blade** (default): Uses Blade templates styled with Tailwind CSS.
   - **Livewire**: Builds reactive UI components in PHP without JavaScript frameworks.
   - **Inertia (Vue/React)**: Integrates Vue or React for SPA-like functionality within the Laravel app.
   - **API**: Provides routes and controllers without views, ideal for headless applications. It also adds a `FRONTEND_URL` to `.env`.

4. Migrate the database and compile assets:
   
   ```shell
   php artisan migrate
   npm install
   npm run dev
   ```

## b. Jetstream:

Laravel **Jetstream** is a more advanced alternative to Breeze. It includes all Breeze features plus:

- Team management (optional).
- Two-factor authentication.
- Session management.

Jetstream supports Blade and Inertia stacks, offering robust tools for larger, more complex applications.

# 4. Sanctum

Laravel Sanctum provides an authentication system for SPAs, mobile apps, and token-based APIs. It supports both token-based and session-based authentication.

## a. Installation:

- Laravel 10 includes Sanctum by default, so no installation is needed.

- For Laravel 11, the API services are not included by default. You can install them via:

```shell
php artisan install:api
```

Alternatively, install Sanctum using Composer:

```shell
composer require laravel/sanctum
```

After installation, publish and configure Sanctum's resources:

```shell
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
```

## b. Token-Based Authentication

Tokens are ideal for authenticating mobile apps and third-party APIs. For first-party SPAs, session-based authentication is recommended.

#### => Issuing API Tokens

1. Ensure your `User` model uses the `HasApiTokens` trait to enable token-related methods.

2. Use the `createToken()` method to generate a token:

```php
$token = $user->createToken('Token Name');
return $token->plainTextToken;
```

    3. This stores the hashed token in the `personal_access_tokens` table. Use the         `plainTextToken` property to access its raw value.

    4. Include the token in API requests using the **Authorization** header:

```js
Authorization: Bearer <token>
```

#### => Managing Tokens

- Retrieve all tokens for a user:
  
  ```php
  foreach ($user->tokens as $token) {
      // Process tokens
  }
  ```

- Protect routes using the `auth:sanctum` middleware:
  
  ```php
  Route::get('/users', [UserController::class, 'index'])->middleware('auth:sanctum');
  ```

#### => Abilities

Sanctum allows you to assign "abilities" (permissions) to tokens:

- Assign abilities during token creation:
  
  ```php
  $token = $user->createToken('Token Name', ['edit-post']);
  ```

- Check token abilities in requests:
  
  ```php
  if ($user->tokenCan('edit-post')) {
      // Authorized action
  }
  ```

- Protect routes with ability-based middleware:
  
  ```php
  Route::middleware(['auth:sanctum', 'abilities:edit-post,add-post'])->get('/posts', ...);
  ```
  
  - **`abilities`**: Verifies if the token has all listed abilities.
  
  - **`ability`**: Verifies if the token has at least one of the listed abilities.

#### => Revoking Tokens

- Revoke all tokens for a user:
  
  ```php
  $user->tokens()->delete();
  ```

- Revoke the current request token:
  
  ```php
  $user->currentAccessToken()->delete();
  ```

- Revoke a specific token:
  
  ```php
  $user->tokens()->where('name', $tokenName)->delete();
  ```

#### => Token Expiration

- Tokens do not expire by default. Set expiration globally in `config/sanctum.php`:
  
  ```php
  'expiration' => 525600, // In minutes (1 year)
  ```

- Set expiration per token:
  
  ```php
  $user->createToken('Token Name', ['*'], now()->addMonth());
  ```

## c. Session-Based Authentication for SPAs:

For SPAs, Sanctum uses session cookies instead of tokens.

##### => Same Top-Level Domain

Sanctum uses secure **HttpOnly** cookies for session management. To function correctly, the front-end and back-end should share the same top-level domain (e.g., `api.domain.com` and `domain.com`).

Configure this in `config/session.php`:

```php
'domain' => 'localhost', // No ports or slashes
```

#### => CORS Configuration

Set up CORS headers in `config/cors.php`:

```php
'paths' => ['*'],
'allowed_origins' => [env('FRONTEND_URL')],
'supports_credentials' => true,
```

#### => Stateful Domains

Define stateful domains in `config/sanctum.php`:

```php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost:3000')),
```

#### => SPA Authentication Workflow

1. **Initialize CSRF Protection**:
   
   Make a GET request to `/sanctum/csrf-cookie` to initialize CSRF protection. This sets an `XSRF-TOKEN` cookie.
   
   ```js
   axios.defaults.withCredentials = true;
   await axios.get('/sanctum/csrf-cookie');
   ```

2. **Login**:
   
   Submit credentials via a POST request to `/login`:
   
   ```js
   await axios.post('/login', { email: 'user@example.com', password: 'secret' });
   ```

3. **Subsequent Requests**:
   
   Authenticated requests will use the session cookie automatically.
   
   ```js
   const response = await axios.get('/api/posts');
   ```

**Notes**:

- Always call `/sanctum/csrf-cookie` before POST, PUT, or DELETE requests if the user is unauthenticated.
- Define authentication routes in `web.php` for session-based auth instead of `api.php`.

# 5. Socialite

Laravel provides a simple way to authenticate with OAuth providers using the **Socialite** package, in addition to typical form-based authentication.

## a. What is OAuth?

OAuth, short for **Open Authentication**, is a standard for secure access delegation. It allows websites to access user data from other platforms without sharing passwords.

**Note:** OAuth is a protocol, while **Auth0** is a company or SaaS offering OAuth-related services.

Socialite supports authentication via providers like **Facebook**, **Twitter**, **LinkedIn**, **Google**, **GitHub**, **GitLab**, and more.

## b. Installation

Install Socialite using Composer:

```shell
composer require laravel/socialite
```

## c. Configuration

Before using Socialite, you'll need credentials for your OAuth provider (e.g., `client_id`, `client_secret`). These can usually be retrieved by creating a **developer account** on the provider's platform.

#### => Example: GitHub

1. Go to `Settings` > `Developer Settings` > `OAuth Apps`.

2. Set:
   
   - **Homepage URL:** `http://localhost:8000`
   - **Callback URL:** `http://localhost:8000/auth/github/callback`

#### => Example: Google

1. Open the **Google Cloud Console**.

2. Navigate to `APIs & Services` > `Credentials` and create an OAuth credential.

3. Set:

        **Authorized JavaScript origins:** `http://localhost:8000`

        **Redirect URI:** `http://localhost:8000/auth/google/callback`

#### => Store credentials in `.env`

```php
GITHUB_CLIENT_ID=your-client-id
GITHUB_CLIENT_SECRET=your-client-secret

GOOGLE_CLIENT_ID=your-client-id
GOOGLE_CLIENT_SECRET=your-client-secret
```

#### => Update `config/services.php`

Configure the credentials for each provider:

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => 'http://127.0.0.1:8000/auth/github/callback',
],

'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect' => 'http://127.0.0.1:8000/auth/google/callback',
],
```

## c. Routes

To authenticate a user with an OAuth provider, you need **two routes**:

1. **Redirect route:** Redirects the user to the OAuth provider.
2. **Callback route:** Receives the callback and processes the authentication.

#### => Example: GitHub Authentication

- **Redirect Route**

    Redirect the user to the provider using `Socialite::driver()->redirect()`:

```php
Route::get('auth/github/redirect', function () {
    return Socialite::driver('github')->redirect();
}); 
```

- **Callback Route**

    Handle the callback and log in or register the user:

```php
    Route::get('auth/github/callback', function () {
    $ghUser = Socialite::driver('github')->user();

    // Find or create a user
    $user = User::updateOrCreate(
        ['github_id' => $ghUser->id],
        [
            'name' => $ghUser->name,
            'email' => $ghUser->email,
        ]
    );

    // Log the user in
    Auth::login($user);

    // Redirect to dashboard
    return redirect('/dashboard');
});
```

## **Important Notes**

- **Provider Data Differences:** Each OAuth provider may return different user attributes. For example:
  - GitHub provides a **nickname** field.
  - Google does not include a **nickname** but has other fields like **given_name**.
- **Email Conflicts:** If a user logs in using one provider (e.g., GitHub) and tries logging in again using another provider (e.g., Google) with the same email, ensure your logic avoids duplicate accounts.
- **Handle Missing Data:** Not all providers return all user details (e.g., some may not return an email if it's private). Always validate and handle such cases.
