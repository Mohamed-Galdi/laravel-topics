# Mastering Large Datasets in Laravel

Handling large datasets in Laravel—think millions of records—requires careful planning to ensure your application remains fast, scalable, and memory-efficient. Whether you're building reports, processing API data, or generating exports, this guide provides **practical, Laravel-specific techniques** to optimize database operations. We'll cover seeding test data, efficient querying, processing large datasets, caching, and scaling strategies, all with real-world examples.

## 1. Seeding Large Datasets for Testing

To test performance with large datasets, you need realistic test data. Let’s create an Artisan command to seed **1 million user records** efficiently, using batch inserts and transactions to minimize overhead.

Create a command with:

```bash
php artisan make:command seed
```

Here’s the optimized code:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class seed extends Command
{
    protected $signature = 'seed:users {--batch-size=10000} {--total=1000000}';
    protected $description = 'Efficiently seed database with users (single process)';

    public function handle()
    {
        $batchSize = (int) $this->option('batch-size');
        $total = (int) $this->option('total');

        $this->info("Starting to seed {$total} users in batches of {$batchSize}");

        // Optimize for bulk inserts
        ini_set('memory_limit', '2G');
        DB::connection()->disableQueryLog();

        $hashedPassword = Hash::make('password');
        $created = 0;
        $skipped = 0;
        $bar = $this->output->createProgressBar($total);
        $bar->setFormat('%current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s% %memory:6s%');

        $startTime = microtime(true);

        while ($created < $total) {
            $currentBatch = min($batchSize, $total - $created);

            // Generate batch data with simple timestamp
            $batch = [];

            for ($i = 0; $i < $currentBatch; $i++) {
                $name = fake()->name();
                $batch[] = [
                    'name' => $name,
                    'email' => Str::slug($name) . ($created + $i) . '@example.com',
                    'password' => $hashedPassword,
                    'created_at' => fake()->dateTimeThisYear(),
                ];
            }

            // Insert in chunks with error handling
            $chunks = array_chunk($batch, 2000);

            foreach ($chunks as $chunk) {
                try {
                    DB::table('users')->insert($chunk);
                } catch (\Exception $e) {
                    // Try smaller chunks
                    $smallerChunks = array_chunk($chunk, 100);
                    foreach ($smallerChunks as $smallChunk) {
                        try {
                            DB::table('users')->insert($smallChunk);
                        } catch (\Exception $smallError) {
                            $skipped += count($smallChunk);
                        }
                    }
                }
            }

            $created += $currentBatch;
            $bar->advance($currentBatch);

            // Memory cleanup
            unset($batch, $chunks);
            if ($created % 50000 === 0) {
                gc_collect_cycles();
                $elapsed = microtime(true) - $startTime;
                $rate = $created / $elapsed;
                $this->info("\nProcessed {$created}/{$total} users. Rate: " . round($rate, 2) . " users/sec. Skipped: {$skipped}");
            }
        }

        $bar->finish();
        $this->newLine();

        $elapsed = microtime(true) - $startTime;
        $rate = $created / $elapsed;
        $this->info("Completed: {$created} users processed in " . round($elapsed, 2) . " seconds");
        $this->info("Average rate: " . round($rate, 2) . " users/second");
        if ($skipped > 0) {
            $this->warn("Skipped {$skipped} records due to errors");
        }

        // Verify actual count
        $actualCount = DB::table('users')->count();
        $this->info("Total users in database: {$actualCount}");
    }
}
```

To insert 200,000 users in batches of 10,000:

```bash
php artisan seed:users --total=200000 --batch-size=10000
```

Example output:

![](C:\Users\DELL\Desktop\projects\others\laravel-topics\10-mastering-large-datasets-in-laravel\seeding-output.webp)

> ⚠️ **Note**: Some records may be skipped (about 2%) due to timezone-related `DateTime` issues when generating invalid timestamps. This is expected and safe to ignore for testing purposes.

> 📝 Also, make sure `created_at` is included in the `$fillable` property of your `User` model.

**Final Tip**

It's not recommended to insert **1 million users in one go**, especially on limited hardware. Instead, consider splitting it into smaller runs—like 200,000 users per run—and execute the command multiple times.

## 2. Querying Large Datasets Efficiently

Querying millions of records can strain memory and CPU. Here are optimized approaches, benchmarked against a 1-million-user dataset.

### a. Query Methods Compared

Let’s retrieve users created after April 3, 2025, using different methods.

#### Eloquent ORM

```php
use App\Models\User;
use Illuminate\Support\Benchmark;

Route::get('/eloquent', function () {
    [$value, $duration] = Benchmark::value(fn () => 
        User::where('created_at', '>', '2025-04-03')->get()
    );

    return [
        'method' => 'eloquent',
        'duration_ms' => $duration,
        'total_results' => $value->count(),
    ];
});
```

**Output**: ~7500ms, 600,000+ results.

**Why It’s Slow**: Eloquent hydrates each row into a model, consuming significant memory. Avoid for large result sets unless using pagination or chunking.

#### Query Builder (DB Facade)

```php
use Illuminate\Support\Facades\DB;

Route::get('/db-facade', function () {
    [$value, $duration] = Benchmark::value(fn () => 
        DB::table('users')->where('created_at', '>', '2025-04-03')->get()
    );

    return [
        'method' => 'db-facade',
        'duration_ms' => $duration,
        'total_results' => $value->count(),
    ];
});
```

**Output**: ~3000ms, 600,000+ results.

**Why It’s Faster**: Returns `stdClass` objects, skipping Eloquent’s overhead. Better for large reads but still loads all data into memory.

#### Raw SQL

```php
Route::get('/raw-sql', function () {
    [$value, $duration] = Benchmark::value(fn () => 
        DB::select('SELECT * FROM users WHERE created_at > :date', ['date' => '2025-04-03'])
    );

    return [
        'method' => 'raw-sql',
        'duration_ms' => $duration,
        'total_results' => count($value),
    ];
});
```

**Output**: ~2000ms, 600,000+ results.

**Why It’s Fastest**: Minimizes Laravel overhead, using direct PDO queries. Ideal for read-only operations.

**When to Use**:

- **Eloquent**: For small datasets or when model features (e.g., mutators) are needed.
- **Query Builder**: For large reads without Eloquent features.
- **Raw SQL**: For maximum performance in read-heavy scenarios.

### b. Select Only Needed Columns

Fetching fewer columns reduces data transfer and memory usage. Example:

```php
Route::get('/raw-sql-select', function () {
    [$value, $duration] = Benchmark::value(fn () => 
        DB::select('SELECT name, created_at FROM users WHERE created_at > :date', ['date' => '2025-04-03'])
    );

    return [
        'method' => 'raw-sql-select',
        'duration_ms' => $duration,
        'total_results' => count($value),
    ];
});
```

**Output**: ~1500ms, 600,000+ results.

**Why It’s Faster**: Less data is transferred and parsed. Always select only the columns you need.

### c. Indexing for Speed

An index in MySQL is a data structure that improves the speed of data retrieval operations on database tables. It works similarly to an index in a book, allowing the database to locate and access rows quickly without scanning the entire table.

##### How It Works:

For example, if we have a table of users and want to search for a user with the name "Xavi," without an index, MySQL performs a sequential scan, checking each row one by one until it finds a match.

When we add an index on the **username** column, MySQL organizes usernames in a sorted data structure (typically a B-tree) and can use binary search algorithms to locate "Xavi" more efficiently.

![](C:\Users\DELL\Desktop\projects\others\laravel-topics\10-mastering-large-datasets-in-laravel\database-indexes.webp)

##### Pros of Using Indexes:

- **Speeds Up Queries**: Indexes reduce data retrieval time by allowing MySQL to find the needed rows faster.
- **Efficient Sorting and Filtering**: With an index, `ORDER BY` and `GROUP BY` operations run faster.
- **Improved Join Performance**: Indexed foreign keys make joining tables faster.

##### Cons of Using Indexes:

- **Storage Space**: Indexes require additional storage, which can be significant in very large databases.
- **Slower Writes**: `INSERT`, `UPDATE`, and `DELETE` operations take longer because MySQL must update the indexes to reflect changes in the table.
- **Maintenance**: Indexes need occasional maintenance and optimization, especially for tables with frequent updates.

##### What Columns Should Be Indexed?

**Good candidates for indexing:**

- Columns frequently used in `WHERE`, `ORDER BY`, or `JOIN` clauses (e.g., `created_at`, `email`, `user_id`)

- Foreign key columns (e.g., `post_id` in `comments`)

- Unique or semi-unique identifiers (e.g., `slug`, `username`)

**Columns you should avoid indexing:**

- **High-cardinality text columns** like `description` or `bio` (long `TEXT` or `LONGTEXT` types)

- **Boolean columns** or **very low-cardinality fields** (e.g., `is_active`) — MySQL will scan rows anyway

- Frequently updated columns (to avoid index update cost)

##### Index Creation:

raw sql:

```sql
CREATE INDEX idx_users_created_at ON users(created_at);
```

laravel migration:

```php
Schema::table('users', function (Blueprint $table) {
    $table->index('created_at', 'idx_users_created_at');
});
```

> Note: `PRIMARY KEY` and `UNIQUE` columns are automatically indexed by MySQL.

Re-run the raw SQL query:

```php
Route::get('/raw-sql-indexed', function () {
    [$value, $duration] = Benchmark::value(fn () => 
        DB::select('SELECT name, created_at FROM users WHERE created_at > :date', ['date' => '2025-04-03'])
    );

    return [
        'method' => 'raw-sql-indexed',
        'duration_ms' => $duration,
        'total_results' => count($value),
    ];
});
```

**Output**: ~1100ms, 600,000+ results.

**Why It’s Faster**: The index allows MySQL to jump to relevant rows.

**When to Index**:

- Columns in `WHERE`, `ORDER BY`, or `JOIN` clauses (e.g., `created_at`, `email`).
- Foreign keys (e.g., `user_id` in related tables).
- Avoid indexing low-cardinality (e.g., booleans) or frequently updated columns.

### d. Using `EXPLAIN` to Analyze Queries

Adding an index is **not enough** — just because an index exists doesn't mean MySQL will use it.  
To truly understand how your query is executed, you need to inspect the **query plan**. That’s where `EXPLAIN` comes in.

`EXPLAIN` lets you see **how MySQL plans to execute your query**:

- Will it use an index?

- Will it scan the whole table?

- How many rows does it expect to read?

- What join strategies will it use?

If a query is slow, `EXPLAIN` is your **go-to tool** to find out **why**.

##### How to Use EXPLAIN

In Raw SQL:

```php
DB::select('EXPLAIN SELECT name, created_at FROM users WHERE created_at > ?', ['2025-04-03']);
```

In Laravel (Query Builder):

```php
DB::table('users')->where('created_at', '>', '2025-04-03')->explain();
```

Output example:

```php
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "users",
    "type": "ALL",
    "possible_keys": "idx_users_created_at",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 1140026,
    "Extra": "Using where"
  }
]
```

##### EXPLAIN Output

| Field           | Meaning                                                                     | What to Look For                                                     |
| --------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `id`            | The order of execution. 1 = simple query                                    | Usually 1 for simple SELECTs                                         |
| `select_type`   | Type of SELECT (e.g., SIMPLE, SUBQUERY, DERIVED, etc.)                      | SIMPLER is better (means no subqueries)                              |
| `table`         | The table being queried                                                     | Should be the expected table                                         |
| `type`          | 🔥 **Join type** — shows how MySQL reads rows                               | The most important column in this list                               |
| `possible_keys` | Indexes that *could* be used                                                | If it's `NULL`, no index is usable                                   |
| `key`           | Index actually **used** for this query                                      | If `NULL`, MySQL didn’t use any index                                |
| `key_len`       | Length of the key used                                                      | Longer is usually better (up to a point)                             |
| `ref`           | What column or constant is compared against the index                       | Useful for joins and foreign keys                                    |
| `rows`          | Estimated number of rows to scan                                            | Lower is better                                                      |
| `Extra`         | Additional notes like `Using index`, `Using where`, `Using temporary`, etc. | `Using where` is common; avoid `Using filesort` or `Using temporary` |

##### `type`: The Most Important Column

This column tells you **how efficient** your scan is. Here's how the values rank from **worst to best**:

| `type` value       | Meaning                                      | Performance |
| ------------------ | -------------------------------------------- | ----------- |
| `ALL`              | Full table scan                              | ❌ Worst     |
| `index`            | Full index scan                              | ⚠️ Medium   |
| `range`            | Index range scan (e.g., `>`, `<`, `BETWEEN`) | ✅ Good      |
| `ref`              | Index lookup with a constant key             | ✅ Better    |
| `eq_ref`           | One row per key (used in joins)              | ✅✅ Great    |
| `const` / `system` | Only one row matched                         | 🏆 Best     |

## 3. Processing Large Datasets

Fetching large datasets into memory can crash your app. Use these techniques to process data efficiently.

### a. Chunking Results

Process records in smaller chunks to reduce memory usage:

```php
Route::get('/chunk', function () {
    $start = microtime(true);
    User::where('created_at', '>', '2025-04-03')->chunk(1000, function ($users) {
        foreach ($users as $user) {
            // Process each user (e.g., update or export)
        }
    });
    return ['duration_ms' => (microtime(true) - $start) * 1000];
});
```

**Why It Works**: Loads only 1000 records at a time, freeing memory after each chunk.

### b. Lazy Loading

For even lower memory usage, use lazy loading:

```php
Route::get('/lazy', function () {
    $start = microtime(true);
    User::where('created_at', '>', '2025-04-03')->lazy(1000)->each(function ($user) {
        // Process each user
    });
    return ['duration_ms' => (microtime(true) - $start) * 1000];
});
```

**Why It’s Better**: Streams results directly from the database cursor, minimizing memory.

### c. Queueing for Background Processing

For time-consuming tasks (e.g., exporting data), use Laravel’s queue system:

```php
use App\Jobs\ProcessUserBatch;

Route::get('/queue', function () {
    User::where('created_at', '>', '2025-04-03')->chunk(1000, function ($users) {
        ProcessUserBatch::dispatch($users);
    });
    return ['status' => 'Processing queued'];
});
```

Job example:

```php
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class ProcessUserBatch implements ShouldQueue
{
    use Dispatchable, Queueable;

    protected $users;

    public function __construct($users)
    {
        $this->users = $users;
    }

    public function handle()
    {
        foreach ($this->users as $user) {
            // Process user (e.g., generate report)
        }
    }
}
```

**When to Use**:

- **Chunking**: For in-memory processing with moderate memory constraints.
- **Lazy Loading**: For minimal memory usage with streaming.
- **Queueing**: For long-running tasks or offloading to background workers.

## 4. Don’t Forget Caching

Even the most optimized query can become a bottleneck if it's executed too often — especially on large datasets. That’s where **caching** comes in.

By caching the results of expensive queries (or even parts of them), you can dramatically reduce database load and improve response times for your users.

Laravel makes caching easy with built-in support for drivers like Redis, Memcached, and file-based storage. It’s one of the **highest-impact optimizations** for read-heavy applications.

> 🧠 Want to dive deeper?  
> Read this guide: [Caching in Laravel

## 5. Scaling Strategies

For very large datasets, consider these advanced techniques:

- **Partitioning**: Split tables by range (e.g., `created_at` by year). Example migration:
  
  ```php
  Schema::create('users', function (Blueprint $table) {
      $table->id();
      $table->string('name');
      $table->string('email')->unique();
      $table->string('password');
      $table->timestamps();
  });
  DB::statement('ALTER TABLE users PARTITION BY RANGE (YEAR(created_at)) (
      PARTITION p2023 VALUES LESS THAN (2024),
      PARTITION p2024 VALUES LESS THAN (2025),
      PARTITION p2025 VALUES LESS THAN (2026)
  )');
  ```

- **Archiving**: Move old data to an archive table to reduce main table size.

- **Connection Pooling**: Use tools like MySQL Proxy or PgBouncer for high-concurrency apps.

- **Sharding**: Distribute data across multiple databases (requires custom logic or tools like Vitess).

**When to Use**: Partitioning for date-based queries; archiving for old data; sharding for horizontal scaling.

#### Best Practices Summary

1. **Always use chunking methods** (`chunk()`, `lazy()`, `cursor()`) for large datasets
2. **Create appropriate indexes** on frequently queried columns
3. **Select only required columns** to reduce memory usage
4. **Use Query Builder or raw SQL** for read-only operations
5. **Implement caching** for frequently accessed data
6. **Monitor query performance** with EXPLAIN and profiling tools
7. **Use database transactions** for bulk operations
