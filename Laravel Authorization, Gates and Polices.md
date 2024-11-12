# Table of Contents

1. [Intro](#1-intro)
2. [What is Authorization?](#2-what-is-authorization)
3. [Gates vs Polices](#3-gates-vs-polices)
4. [Gates](#4-gates)
    - [a. Definition](#a-definition)
    - [b. Usage](#b-usage)
    - [c. Gate Responses](#c-gate-responses)
5. [Policies](#5-policies)
    - [a. Generate a policy](#a-generate-a-policy)
    - [b. Naming conventions](#b-naming-conventions)
    - [c. Writing policies](#c-writing-policies)
    - [d. Methods Without Models](#d-methods-without-models)
    - [e. Policy Responses](#e-policy-responses)
    - [f. Usage](#f-usage)
        - [Via the User Model](#via-the-user-model)
        - [Via the Gate Facade](#via-the-gate-facade)
        - [Via Middleware](#via-middleware)
        - [Via Blade Template](#via-blade-template)
6. [Authorization & Inertia](#6-authorization--inertia)

# 1. Intro

Imagine a blog platform where users can create, view, edit, and delete their posts. Each user's dashboard displays a list of their posts, each with a "View" button leading to a page like `posts/post/1`. However, with this setup, a curious user might try modifying the URL to access posts they didn’t create by guessing routes like `posts/post/4` or `posts/post/500`.

To secure this setup, consider two key strategies:

1. **UUIDs for IDs** – Replace sequential numeric IDs with UUIDs (Universally Unique Identifiers). UUIDs make it much harder for users to guess the URLs of other posts, so a path would appear as `posts/post/550e8400-e29b-41d4-a716-446655440000`.

2. **Authorization Checks** – Even with UUIDs, authorization is essential. Proper authorization ensures that users can only view, edit, or delete posts they own, blocking attempts to access other users' posts directly.

# 2. What is Authorization?

**Authorization** is the process of defining what authenticated users are allowed to do. While **authentication** verifies a user’s identity, **authorization** defines their permissions—determining what actions they can perform on specific resources.

In the context of the blog platform, for example, users may be able to view all posts but only edit or delete posts they own.

Laravel provides an elegant solution for managing authorization through two main methods: **Gates** and **Policies**. These tools allow you to define and enforce permissions in a clean, consistent way across your application.

# 3. Gates vs Polices

In Laravel, **Gates** and **Policies** handle authorization, each with a different scope:

- **Gates** are closures used for quick, single-purpose permissions.
- **Policies** are classes that group related permissions for a specific model.

Think of **Gates** and **Policies** like **routes** and **controllers**:

- **Gates** are similar to defining a route with a closure, handling a single action.
- **Policies** are like controllers, organizing multiple actions related to a model.

### Gate Example

A Gate defines a single permission using a closure, like this:

```php
Gate::define('view-admin-dashboard', function ($user) {
    return $user->is_admin;
});
```

This is similar to a route with a closure:

```php
Route::get('/admin', function () {
    return view('admin.index')
});
```

### Policy Example

A Policy organizes multiple permissions for a model, similar to how a controller groups routes:

```php
class PostPolicy
{
    public function update(User $user, Post $post) {
        return $user->id === $post->user_id;
    }
}
```

This resembles a controller that organizes actions for a resource:

```php
class PostController
{
    public function update(Request $request, Post $post) {
        $post->update($request->all());
        return redirect()->back();
     }
}
```

You do not need to choose between exclusively using gates or exclusively using policies when building an application. Most applications will most likely contain some mixture of gates and policies, and that is perfectly fine! Gates are most applicable to actions that are not related to any model or resource, such as viewing an administrator dashboard. In contrast, policies should be used when you wish to authorize an action for a particular model or resource.

# 4. Gates

## a. Definetion:

 Gates are defined within the `boot` method of the `App\Providers\AppServiceProvider` class using the `Gate` facade. 

To create a Gate we use the **Gate** facade with the `define` method. This method takes two arguments:

1. The **ability name** (a unique identifier for the Gate).
2. A **closure** that performs the authorization check, typically returning a boolean.

Example:

```php
// App\Providers\AppServiceProvider.php
use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;


public function boot(): void
{
    Gate::define('update-post', function (User $user, Post $post) {
        return $user->id === $post->user_id;
    });
}
```

In this example, `'update-post'` is the **ability name**, and the closure checks if the user is the owner of the post.

Like routes, gates may also be defined using a class callback array:

```php
public function boot(): void
{
    Gate::define('update-post', [PostPolicy::class, 'update']);
}
```

## b. Usage:

To define a Gate, we used the `define` method of the **Gate** facade. For usage, we rely on the `allows` or `denies` methods.

These methods take the **ability name** as the first argument and the resource as the optional second argument.

Note: You don’t need to explicitly pass the authenticated user to these methods—Laravel will automatically provide the current user to the Gate closure.

Example (using the gate in an `update` method of a controller):

```php
class PostController extends Controller
{
    public function update(Request $request, Post $post): RedirectResponse
    {
        if (! Gate::allows('update-post', $post)) {
            abort(403);
        }

        // Update the post...

        return redirect('/posts');
    }
}
```

💡 **Note**: `abort()` is a helper provided by Laravel to generate HTTP exceptions from anywhere in your application.

If you want to check permissions for a specific user rather than the authenticated one, you can use the `forUser()` method:

```php
Gate::forUser($user)->allows('update-post', $post)
Gate::forUser($user)->denies('update-post', $post)
```

You may also authorize multiple actions at once using the `any` or `none` methods:

```php
Gate::any(['update-post', 'delete-post'], $post)
Gate::none(['update-post', 'delete-post'], $post)
```

## c. Gate Responses:

So far, we’ve only examined Gates that return simple boolean values. Sometimes, though, you may want to return a more detailed response, including an error message. To do so, return an `Illuminate\Auth\Access\Response` from the gate:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
                ? Response::allow()
                : Response::deny('You must be an administrator.');
});
```

Even when returning a response from your gate, the `Gate::allows` method still returns a boolean value. However, the `Gate::inspect` method can retrieve the full authorization response:

```php
$response = Gate::inspect('edit-settings');

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

By default, a denied Gate returns a `403` HTTP response, but you can customize the HTTP status code with the `denyWithStatus` static constructor:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
                ? Response::allow()
                : Response::denyWithStatus(404);
});
```

To return a `404` (not found) response for unauthorized actions—a common pattern for hiding resources—the `denyAsNotFound` method is available:

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
                ? Response::allow()
                : Response::denyAsNotFound();
});
```

All Gates should be defined in the `boot()` method of your **App Service Provider**, the same place we define other global configurations, like `Model::shouldBeStrict()`. For more complex authorization requirements, this method can get crowded, which is why Policies are often a better choice for organizing permissions related to specific models.

# 5. Policies

## a. Generate a policy:

You can generate a policy using the `make:policy` Artisan command, which will place the policy in the `app/Policies` directory:

```shell
php artisan make:policy PostPolicy
```

This command generates an empty policy class. However, using the `--model` option includes methods for actions like `viewAny`, `view`, `create`, `update`, `delete`, `restore`, and `forceDelete`:

```shell
php artisan make:policy PostPolicy --model=Post
```

## b. Naming conventions:

Policies should be located in a `Policies` directory within or above your models’ directory. Laravel will look for policies in `app/Models/Policies` and `app/Policies`.

If the model and policy follow Laravel’s naming conventions—where the policy name matches the model name with a `Policy` suffix (e.g., `User` model and `UserPolicy`)—Laravel will automatically discover the policy.

## c. Writing policies:

After creating a policy, add methods for each action you want to authorize. For instance, the `update` method in `PostPolicy` determines if a `User` can update a `Post`.

The `update` method takes a `User` and `Post` instance as arguments and should return `true` or `false` based on whether the user is authorized. Here, it verifies that the user's `id` matches the `post`'s `user_id`:

```php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
}
```

## d. Methods Without Models:

Some policy methods only need an instance of the authenticated user, especially for actions like `create`. For example, when creating a blog, you may want to check if a user is authorized to create posts in general. In such cases, the policy method should only accept a `User` instance:

```php
public function create(User $user): bool
{
    return $user->role == 'writer';
}
```

## e. Policy Responses:

Similar to **Gates**, policies allow control over the returned response. You can return a detailed response with an error message by using an `Illuminate\Auth\Access\Response` instance:

```php
public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
                ? Response::allow()
                : Response::deny('You do not own this post.');
}
```

Alternatively, you can specify a different HTTP status:

```php
public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
                ? Response::allow()
                : Response::denyWithStatus(404);
}
```

## f. Usage:

#### => Via the User Model

Laravel’s `User` model includes `can` and `cannot` methods for authorizing actions. These methods take the action name as the first argument and the relevant model as the second.

For example, within a controller method, you can determine if a user can update a given `Post` model:

```php
class PostController extends Controller
{
    public function update(Request $request, Post $post)
    {
        if ($request->user()->cannot('update', $post)) {
            abort(403);
        }

        // Update the post...

        return redirect('/posts');
    }
}
```

Some actions, like `create`, don’t require a model instance. In these cases, you can pass the class name to `can` or `cannot`, and Laravel will determine the appropriate policy:

```php
if ($request->user()->cannot('create', Post::class)) {
    abort(403);
}
```

#### => Via the `Gate` Facade

In addition to the methods on the `User` model, you can authorize actions using the `Gate` facade’s `authorize` method:

```php
class PostController extends Controller
{
    public function update(Request $request, Post $post)
    {
        Gate::authorize('update', $post);

        // The current user can update the blog post...

        return redirect('/posts');
    }
}
```

For actions that don’t require models:

```php
Gate::authorize('create', Post::class);
```

#### => Via Middleware

Laravel includes middleware for authorizing actions before requests reach routes or controllers. Use the `can` middleware alias to authorize a user for an action:

```php
Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->middleware('can:update,post');
```

In this case, since we are using **implicit model binding**, an `App\Models\Post` model will be passed to the policy method.

For convenience, you may also attach the `can` middleware to your route using the `can` method:

```php
Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->can('update', 'post');
```

And for actions that don't require models:

```php
Route::post('/post', function () {
    // The current user may create posts...
})->can('create', Post::class);
```

#### => Via Blade Template

When writing Blade templates, you may want to display content based on user permissions. Use the `@can` and `@cannot` directives to conditionally render content:

```php
@can('update', $post)
    <!-- The current user can update the post... -->
@elsecan('create', App\Models\Post::class)
    <!-- The current user can create new posts... -->
@else
    <!-- ... -->
@endcan

@cannot('update', $post)
    <!-- The current user cannot update the post... -->
@elsecannot('create', App\Models\Post::class)
    <!-- The current user cannot create new posts... -->
@endcannot
```

💡 **Note**: Client-side authorization is not sufficient on its own; server-side checks are also necessary.

# 6. Authorization & Inertia

While authorization checks should always occur on the server, it can be helpful to expose authorization data to the frontend to control the UI based on user permissions. Laravel doesn’t enforce a specific way to pass authorization information to an Inertia-powered frontend.

However, if you’re using one of Laravel’s Inertia starter kits, your application already includes a `HandleInertiaRequests` middleware. This middleware’s `share` method provides shared data accessible to all Inertia pages, making it an ideal place to define and expose authorization information for the user.

Here's an example implementation within the `HandleInertiaRequests` middleware, where the `share` method includes user permissions:

```php
class HandleInertiaRequests extends Middleware
{
    public function share(Request $request)
    {
        return [
            ...parent::share($request),
            'auth' => [
                'user' => $request->user(),
                'permissions' => [
                    'post' => [
                        'create' => $request->user()->can('create', Post::class),
                    ],
                ],
            ],
        ];
    }
}
```

In this example:

- The `auth` object is shared with all Inertia views, providing the currently authenticated user and their permissions.
- The `permissions` array contains a `post` key, which includes authorization for creating a post. This permission can then be accessed by the frontend to adjust the UI accordingly.

This approach allows your Inertia-based frontend to react to user permissions without directly handling sensitive authorization logic on the client side.
