# Table Of Content

- [1. Intro To Cache](#1-intro-to-cache)
- [2. Drivers and Configuration](#2-drivers-and-configuration)
- [3. Why Database is the Default Cache Driver](#3-why-database-is-the-default-cache-driver)
- [4. Cache Usage](#4-cache-usage)
  - [::get()](#get)
  - [::put()](#put)
  - [::has()](#has)
  - [::increment() / ::decrement()](#increment--decrement)
  - [::remember()](#remember)
  - [::forget()](#forget)
  - [::flush()](#flush)
- [5. Impact on Server Resources](#5-impact-on-server-resources)
- [6. Caching Views and Routes in Laravel](#6-caching-views-and-routes-in-laravel)
- [7. Browser Cache](#7-browser-cache)

# 1. Intro To Cache

In Laravel, **caching** is a mechanism for storing frequently accessed data temporarily in a fast storage medium (like memory) to improve the performance of your application. By reducing the need to perform expensive operations (e.g., database queries, API calls, or complex computations), caching can significantly enhance the speed and efficiency of your application.

# 2. Drivers and Configuration

Laravel supports popular caching backends like **<u>Memcached</u>**, **<u>Redis</u>**, **<u>DynamoDB</u>**, and relational **<u>databases</u>** out of the box. In addition, a **<u>file</u>** based cache driver is available, while **<u>array</u>** and **<u>null</u>** cache drivers provide convenient cache backends for your automated tests.

By default, Laravel is configured to use the **<u>database</u>** cache driver, which stores the serialized, cached objects in your application's database.

Your application's cache configuration file is located at `config/cache.php`. In this file, you may specify which cache store you would like to be used by default throughout your application.

```php
'driver' => env('CACHE_DRIVER', 'database'),
```

# 3. Why Database is the Default Cache Driver

Laravel 11 sets the **<u>database</u>** as the default cache driver, which can seem counterintuitive since caching is typically used to reduce database load by storing frequently accessed data in faster storage.

However, using a dedicated `cache` table simplifies data retrieval by avoiding complex or computationally expensive queries. Even though the data is stored in the same database, fetching it from the `cache` table is significantly faster than recalculating or re-fetching it from the main tables.

This choice also offers several benefits:

- **Ease of Setup**: The database is universally available, requiring no extra configuration.
- **Portability**: Works across all environments, even on shared hosting.
- **Data Persistence**: Cached data remains available even after server restarts.

While memory-based caches like Redis provide better performance, the database cache driver ensures Laravel is functional out-of-the-box, especially for smaller projects or those without advanced performance needs.

# 4. Cache Usage

To obtain a cache store instance, you may use the `Cache` facade, which is what we will use throughout this doc.

### ::get()

The `Cache` facade's `get` method is used to retrieve items from the cache. If the item does not exist in the cache, `null` will be returned. If you wish, you may pass a second argument to the `get` method specifying the default value you wish to be returned if the item doesn't exist:

```php
$value = Cache::get('key');

$value = Cache::get('key', 'default');
```

### ::put()

You may use the `put` method on the `Cache` facade to store items in the cache, it take three parameters: **key**, **value**, and **ttl** (in seconds).

```php
Cache::put('key', 'value', 60*10);
```

If the **ttl** (time to live) is not passed to the `put` method, the item will be stored indefinitely.

A common way to set the **TTL (Time-To-Live)** is by using expressions like `60 * 2` instead of directly writing `120`. This approach is more readable and easier to modify. For example, to set it to 5 minutes, you only need to change the `2` to `5`. This method also works for hours and days, such as `60 * 60 * 3` for 3 hours.

Instead of passing the number of seconds as an integer, you may also pass a `DateTime` instance representing the desired expiration time of the cached item:

```php
Cache::put('key', 'value', now()->addMinutes(10));
```

### ::has()

The `has` method may be used to determine if an item exists in the cache. This method will also return `false` if the item exists but its value is `null`:

```php
if (Cache::has('key')) {
    // ...
}
```

### ::increment() / ::decrement()

The `increment` and `decrement` methods may be used to adjust the value of integer items in the cache. Both of these methods accept an optional second argument indicating the amount by which to increment or decrement the item's value:

```php
Cache::increment('key');
Cache::increment('key', $amount);
Cache::decrement('key');
Cache::decrement('key', $amount);
```

If the value is not an integer, PHP (being a loosely typed language) will treat it as zero.

### ::remember()

A common case is to retrieve an item from the cache, but also store a default value if the requested item doesn't exist. we can achieve that using the **::has()**, **::get()** and **::set()** methods, like this:

```php
public function index(){

    if(Cache::has('posts')){
        $posts = Cache::get('posts');
    }else{
        $posts = Post::paginate(10);
        Cache::put('posts', $posts, now()->addHours(6));
    }
}
```

You may do the same using the `Cache::remember` method:

```php
$posts= Cache::remember('posts', now()->addHours(6), function () {
    return DB::table('posts')->paginate(10);
});
```

If the item does not exist in the cache, the closure passed to the `remember` method will be executed and its result will be placed in the cache.

### ::forget()

You may remove items from the cache using the `forget` method:

```php
Cache::forget('key');
```

### ::flush()

You may clear the entire cache using the `flush` method:

```php
Cache::flush();
```

You can achieve the same using this artisan command:

```shell
php artisan cache:clear
```

# 5. Impact on Server Resources

Caching can impact server resources, but the effects depend on the **cache driver** you choose and how caching is implemented. Here's a breakdown of how different aspects of caching affect server resources:

- **Memory-Based Cache Drivers (e.g., Redis, Memcached):**
  
  These drivers store cached data in RAM, making them extremely fast but resource-intensive in terms of memory usage.

- **File-Based Cache Driver:**
  
  File I/O is slower than memory-based caching. If caching large or excessive data, it can fill up disk space and increase read/write overhead on the file system.

- **Database Cache Driver:**
  
  Adds read/write operations to the database, which increases database load and disk I/O.

Regarding **CPU** usage, all cache types require data to be serialized before storing and deserialized when retrieved, which uses CPU resources.

**<u>Minimize the Impact of Caching on Resources:</u>**

- **Set Expiry Times:**
  
  - Always define appropriate expiration times for cached data to prevent overloading resources.

- **Use Tags or Keys for Targeted Cache Management:**
  
  - Group related cache entries and flush them together to avoid unnecessary data retention.

- **Avoid Over-Caching:**
  
  - Only cache data that is expensive to retrieve or compute and is frequently accessed.
  - Avoid caching rarely accessed or lightweight data.

- **Monitor Resource Usage:**
  
  - Regularly monitor the server's memory, CPU, disk, and network usage to ensure that caching is not overburdening the system.

# 6. Caching Views and Routes in Laravel

In Laravel, caching views and routes are optimization techniques that improve the performance of your application by reducing repetitive computations.

### Caching Views

View caching stores compiled Blade templates as plain PHP files on the server, so they don't need to be recompiled every time the application serves a request.

**<u>How it works:</u>**

- When you use Blade templates (`resources/views/*.blade.php`), Laravel compiles them into plain PHP files stored in the `storage/framework/views` directory.

- The compiled views are automatically reused unless the original Blade file is modified.

- Compiled views load faster because the application skips the compilation step.

**<u>Manual View Cache:</u>**

Although view caching is handled automatically, you can force Laravel to recompile views by using the Artisan command:

```shell
php artisan view:cache
```

If you need to clear the cached views:

```shell
php artisan view:clear
```

### Caching Routes

Route caching optimizes the application's route registration process by storing a serialized version of all routes in a single file. This speeds up route resolution, especially in large applications with many routes.

**<u>How it works:</u>**

- The `php artisan route:cache` command generates a single, optimized route file at `bootstrap/cache/routes.php`.
- Instead of dynamically loading and parsing routes from your `routes/web.php` or `routes/api.php` files, Laravel uses the cached file.
- Improves performance by avoiding route registration at runtime.

**<u>Manual Route Cache:</u>**

The following command caches all routes, combining `routes/web.php`, `routes/api.php`, and other route files into a single optimized file.

```shell
php artisan route:cache
```

If you make changes to the routes, you'll need to clear the cache before running the `route:cache` command again:

```shell
php artisan route:clear
```

**<u>Important Notes:</u>**

- If your routes use closures (e.g., `Route::get('/test', function () { return 'Hello'; });`), the `route:cache` command will fail because closures cannot be serialized. Instead, use controller methods:
  
  ```php
  Route::get('/test', [TestController::class, 'index']);
  ```

- Use `route:cache` in production environments for better performance but avoid using it during development since changes to routes won't be immediately reflected without clearing and regenerating the cache.

# 7. Browser Cache

The browser cache is a mechanism where web browsers store static assets (like images, JavaScript files, CSS, fonts, and sometimes API responses) locally on the user's device. This reduces the need to re-download these resources every time the user revisits the same website.

- **Purpose**: Stores static assets (e.g., CSS, JavaScript, images) on the user's device to reduce load times for repeated visits.
- **Scope**: Data is specific to each user's browser and website visit.
- **Access and Control**:
  - Browsers automatically cache certain resources to improve performance, but developers can control this behavior.
  - HTTP headers like `Cache-Control`, `ETag`, and `Expires` are used to manage how and when resources are cached.
- **Developer Configuration**:
  - In Laravel, developers can customize browser caching using middleware, response headers, or asset versioning.
- **Storage**: Cached resources are stored locally in the user's browser storage.
- **Example**: Caching an image or a stylesheet so it doesn't need to be re-downloaded on subsequent page loads.

**<u>Key Difference</u>**

**Laravel Cache**: Improves server-side performance by storing reusable data on the server.

**Browser Cache**: Enhances client-side performance by storing reusable assets locally on the user's device. Both work together to create a faster and more efficient user experience.
