# Table Of Content

- [1. Driver and Configuration](#1-driver-and-configuration)
  - [Available Cache Drivers](#available-cache-drivers)
  - [Configuration Setup](#configuration-setup)
- [2. Cache Usage](#2-cache-usage)
  - [The `get()` Method](#the-get-method)
  - [The `has()` Method](#the-has-method)
  - [The `put()` Method](#the-put-method)
  - [The `remember()` Method](#the-remember-method)
  - [Clear The Cache: `forget()` and `flush()`](#clear-the-cache-forget-and-flush)
  - [Additional Cache Methods](#additional-cache-methods)
- [3. Cache Strategies and Concepts](#3-cache-strategies-and-concepts)
  - [Fundamental Concepts](#fundamental-concepts)
  - [Common Caching Strategies](#common-caching-strategies)
  - [Cache Versioning Strategy](#cache-versioning-strategy)
- [4. Trade-offs of Caching](#4-trade-offs-of-caching)
  - [Speed vs Accuracy](#speed-vs-accuracy)
  - [Memory Usage vs Cache Size](#memory-usage-vs-cache-size)
  - [Cache Hit Ratio vs Eviction Policy](#cache-hit-ratio-vs-eviction-policy)
  - [Data Freshness vs Caching Duration](#data-freshness-vs-caching-duration)
  - [Write Performance vs Read Performance](#write-performance-vs-read-performance) 


Caching is a mechanism for storing frequently accessed data temporarily in fast storage to improve your application's performance. By reducing expensive operations like database queries, API calls, or complex computations, caching can significantly enhance your application's speed and efficiency.

## Drivers and Configuration

Laravel provides a flexible caching system that supports multiple storage backends, each with its own characteristics and use cases. Understanding these drivers helps you choose the right caching strategy for your application.

### Available Cache Drivers

**Database Driver**: Stores cached data in your application's database using a dedicated cache table. While this might seem counterintuitive (using a database to cache database results), it's actually quite effective for many scenarios.

**File Driver**: Stores cached data as files on the server's filesystem. This is useful when you want persistent caching without database overhead, but it's generally slower than memory-based solutions.

**Redis Driver**: A powerful in-memory data structure store that excels at caching. Redis offers advanced features like data persistence, clustering, and pub/sub capabilities. For comprehensive Redis setup and advanced usage patterns, check out our [dedicated Redis in Laravel article](https://blog.galdi.dev/articles/getting-started-with-redis-in-laravel).

**Memcached Driver**: A high-performance, distributed memory caching system. It's purely memory-based and excellent for storing simple key-value pairs across multiple servers.

**DynamoDB Driver**: Amazon's NoSQL database service, useful when you're already using AWS infrastructure and want a managed caching solution.

**Array Driver**: Stores cache data in a PHP array during the request lifecycle. This is primarily used for testing since data doesn't persist between requests.

**Null Driver**: A "fake" cache driver that doesn't actually store anything. Also used primarily for testing when you want to disable caching temporarily.

### Configuration Setup

Your application's cache configuration lives in `config/cache.php`. This file defines all available cache stores and specifies which one to use by default:

```php
'default' => env('CACHE_DRIVER', 'database'),

'stores' => [
    'database' => [
        'driver' => 'database',
        'table' => 'cache',
        'connection' => null,
        'lock_connection' => null,
    ],

    'file' => [
        'driver' => 'file',
        'path' => storage_path('framework/cache/data'),
    ],

    'redis' => [
        'driver' => 'redis',
        'connection' => 'cache',
    ],
    // ... other drivers
],
```

You can also use multiple cache stores simultaneously in your application:

```php
// Use the default cache store
Cache::put('key', 'value', 3600);

// Use a specific cache store
Cache::store('redis')->put('key', 'value', 3600);
Cache::store('file')->put('key', 'value', 3600);
```

## Cache Usage

Laravel's Cache facade provides a clean, expressive API for all caching operations. Let's explore each method in detail with practical examples.

### The `get()` Method

The `get()` method is your primary tool for retrieving cached data:

```php
// Basic retrieval
$value = Cache::get('user_count');

// With default value
$value = Cache::get('user_count', 0);

// With closure as default (executed only if key doesn't exist)
$value = Cache::get('expensive_calculation', function() {
    return $this->performExpensiveCalculation();
});
```

The method returns `null` if the key doesn't exist and no default is provided. This is important to remember when checking for cached values.

### The `has()` Method

Use `has()` to check if a key exists in the cache:

```php
if (Cache::has('user_preferences')) {
    $preferences = Cache::get('user_preferences');
    // Process preferences
} else {
    // Load preferences from database
    $preferences = $this->loadUserPreferences();
    Cache::put('user_preferences', $preferences, 3600);
}
```

**Important note**: `has()` returns `false` if the key exists but its value is `null`. This is different from checking if a key exists with a `null` value.

### The `put()` Method

The `put()` method stores data in the cache with a specified time-to-live (TTL):

```php
// Store for 60 seconds
Cache::put('recent_posts', $posts, 60);

// Store for 10 minutes using readable expression
Cache::put('user_session', $sessionData, 60 * 10);

// Store until specific DateTime
Cache::put('flash_sale', $saleData, now()->addHours(6));
```

**TTL Best Practices:**

Using expressions like `60 * 10` instead of `600` makes your code more maintainable:

- `60 * 5` for 5 minutes
- `60 * 60` for 1 hour
- `60 * 60 * 24` for 1 day
- `60 * 60 * 24 * 7` for 1 week

You can also use Laravel's time helpers:

```php
Cache::put('key', 'value', now()->addMinutes(30));
Cache::put('key', 'value', now()->addHours(2));
Cache::put('key', 'value', now()->addDays(1));
```

### The `remember()` Method

The `remember()` method is arguably the most useful caching method. It combines checking, retrieving, and storing in one elegant operation:

```php
// Basic usage
$posts = Cache::remember('recent_posts', 3600, function() {
    return Post::with('author')
        ->where('published_at', '>', now()->subDays(7))
        ->orderBy('published_at', 'desc')
        ->limit(10)
        ->get();
});
```

This replaces the verbose pattern:

```php
if (Cache::has('recent_posts')) {
    $posts = Cache::get('recent_posts');
} else {
    $posts = Post::with('author')
        ->where('published_at', '>', now()->subDays(7))
        ->orderBy('published_at', 'desc')
        ->limit(10)
        ->get();
    Cache::put('recent_posts', $posts, 3600);
}
```

**The `rememberForever()` Method**

Like `remember()`, but stores the data indefinitely:

```php
$siteSettings = Cache::rememberForever('site_settings', function() {
    return Setting::all()->pluck('value', 'key');
});
```

**The `forever()` Method**

Store data indefinitely (until manually removed):

```php
Cache::forever('app_settings', $settings);
```

Use this sparingly, as it can lead to memory issues if overused.

### Clear The Cache: `forget()` and `flush()`

**Removing Specific Items**

```php
// Remove a specific key
Cache::forget('user_session_' . $userId);

// Remove multiple keys
Cache::forget(['key1', 'key2', 'key3']);
```

**Clearing All Cache**

```php
// Clear all cached data
Cache::flush();
```

You can also use the Artisan command:

```bash
php artisan cache:clear
```

### Additional Cache Methods

**The `add()` Method**

The `add()` method only stores data if the key doesn't already exist. This is an atomic operation, making it safe for concurrent requests:

```php
// Returns true if added, false if key already exists
$wasAdded = Cache::add('user_lock_' . $userId, true, 300);

if ($wasAdded) {
    // We got the lock, proceed with operation
    $this->performUserOperation($userId);
    Cache::forget('user_lock_' . $userId);
} else {
    // Another process is already working on this user
    throw new Exception('Operation already in progress for this user');
}
```

This is particularly useful for implementing locks or ensuring single execution of expensive operations.

**The `pull()` Method**

The `pull()` method retrieves a value and immediately removes it from the cache:

```php
// Get and remove in one operation
$flashMessage = Cache::pull('flash_message');
$temporaryToken = Cache::pull('temp_token', 'default_token');

// Useful for one-time data like CSRF tokens or temporary download links
$downloadLink = Cache::pull('download_' . $fileId);
if ($downloadLink) {
    return redirect($downloadLink);
} else {
    abort(404, 'Download link expired or invalid');
}
```

the **`Increment()` and `Decrement()`** methods

These methods are perfect for counters, statistics, or rate limiting:

```php
// Simple increment/decrement
Cache::increment('page_views');
Cache::decrement('available_slots');

// With custom amounts
Cache::increment('user_points', 10);
Cache::decrement('inventory_count', 5);

// Rate limiting example
$attempts = Cache::increment('login_attempts_' . $ip);
if ($attempts > 5) {
    throw new Exception('Too many login attempts');
}
```

**Important behavior**: If the key doesn't exist, it's created with a value of 0 before the operation. If the value isn't numeric, PHP treats it as 0.

## Cache Strategies and Concepts

Understanding cache strategies is crucial for building high-performance applications. Let's explore the key concepts and patterns that professional developers use.

### Fundamental Concepts

**Cache Hit vs Cache Miss**

A **cache hit** occurs when requested data is found in the cache:

```php
$user = Cache::get('user_' . $id); // Returns user data - CACHE HIT
```

A **cache miss** occurs when requested data is not in cache, requiring the expensive operation:

```php
$user = Cache::get('user_' . $id); // Returns null - CACHE MISS
// Now we need to query the database
$user = User::find($id);
Cache::put('user_' . $id, $user, 3600);
```

**Cache Hit Ratio** is the percentage of requests served from cache. A higher ratio means better performance:

- 90%+ hit ratio: Excellent
- 80-90% hit ratio: Good
- Below 80%: Consider reviewing your caching strategy

**Stale Data**

Stale data refers to cached information that is outdated compared to the source of truth. This is an inherent trade-off in caching:

```php
// User updates their profile
$user->update(['name' => 'New Name']);

// But cached data still shows old name
$cachedUser = Cache::get('user_' . $user->id); // Still has old name
```

Managing stale data requires careful cache invalidation strategies.

**Cache Invalidation**

Cache invalidation is the process of removing or updating cached data when the underlying data changes.

**Eviction Policy**

eviction policy is the rule your cache system uses to decide **which items to remove** when the cache is full.

Most caching systems (like Redis or Memcached) have a limited amount of memory, so when it fills up, they need to evict (delete) something to make space for new data.

### Common Caching Strategies

**Cache-Aside (Lazy Loading)**

This is the most common pattern - check cache first, load from source if missing:

```php
public function getUser($id) {
    return Cache::remember("user_{$id}", 3600, function() use ($id) {
        return User::with('profile', 'preferences')->find($id);
    });
}
```

**Pros**: Simple to implement, only caches requested data **Cons**: First request always slower (cache miss)

**Write-Through Caching**

Update cache immediately when data changes:

```php
public function updateUser($id, array $data) {
    $user = User::find($id);
    $user->update($data);

    // Immediately update cache
    $freshUser = User::with('profile', 'preferences')->find($id);
    Cache::put("user_{$id}", $freshUser, 3600);

    return $user;
}
```

**Pros**: Cache is always up-to-date **Cons**: Extra overhead on writes, cache might store unused data

**Write-Behind (Write-Back) Caching**

Update cache immediately, but defer database writes. This is complex and rarely used in typical web applications, but can be powerful for high-write scenarios.

**Cache Invalidation on Write**

Remove cache when data changes, let next read repopulate:

```php
public function updateUser($id, array $data) {
    $user = User::find($id);
    $user->update($data);

    // Remove from cache - next read will repopulate
    Cache::forget("user_{$id}");

    return $user;
}
```

**Pros**: Ensures no stale data, simple to implement **Cons**: Next read after update will be slower

### Cache Versioning Strategy

Cache versioning is a smart strategy to **invalidate related cache entries all at once** — without deleting each one manually.

Instead of using a fixed cache key like `posts`, you include a version number in the key, like `posts_v1`. When the data changes, you increment the version, and Laravel starts using a **new cache key**, making the old one obsolete.

**Example:**

```php
class PostService {
    private function getVersion() {
        return Cache::rememberForever('posts_version', fn() => 1);
    }

    public function getAllPosts() {
        $version = $this->getVersion();
        return Cache::remember("posts_v{$version}", 3600, function() {
            return Post::with('author')->get();
        });
    }

    public function invalidateCache() {
        Cache::increment('posts_version');
    }
}
```

**How It Works**

1. First time calling `getAllPosts()`  
  → Key used: `posts_v1`  
  → Data is cached for 1 hour
  
2. A post is updated → `invalidateCache()` is called  
  → Version becomes `2`
  
3. Next time `getAllPosts()` is called  
  → Key used: `posts_v2`  
  → New data is fetched and cached
  

✅ The old key (`posts_v1`) still exists but will expire after 1 hour and won't be used again.

## Trade-offs of Caching

Caching improves speed and scalability, but it comes with trade-offs. To make the most of it, you need to balance performance, accuracy, and resource usage. Here are key trade-offs to consider in any Laravel caching strategy:

### Speed vs Accuracy

- **Faster reads** come from serving cached data.
  
- But **data might be outdated** depending on your TTL (time to live) or refresh logic.
  
- For critical or real-time data, short TTLs or cache-busting strategies (like versioning) are better.
  

### Memory Usage vs Cache Size

- **Larger caches** mean faster access but consume more memory (especially in Redis/Memcached).
  
- Watch for unused or outdated keys that stay until they expire.
  
- Use appropriate TTLs to avoid bloating memory.
  

### Cache Hit Ratio vs Eviction Policy

- High **cache hit ratio** improves performance but requires enough memory to keep popular data.
  
- When memory fills up, eviction policies (like LRU – Least Recently Used) decide which keys to remove.
  
- Understand your cache driver's eviction behavior.
  

### Data Freshness vs Caching Duration

- Long durations = fewer database hits, but data might be stale.
  
- Short durations = fresher data, but more backend load.
  
- Use different TTLs based on how often the data changes (e.g., 1 week for countries, 5 minutes for active users).
  

### Write Performance vs Read Performance

- Writes to cache add a bit of overhead (especially for large objects or complex serialization).
  
- Reads are much faster, so it’s usually a worthwhile trade-off — but avoid caching very fast, lightweight queries.
  
- Optimize by caching only what's expensive to compute or query.
  

Caching is not a one-size-fits-all solution — it's a performance tool that must be tuned to your app’s needs.

A smart caching strategy will improve user experience and reduce database load — but only when used with intention and care.