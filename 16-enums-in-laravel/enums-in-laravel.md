# Laravel Enums: Complete Guide to Type-Safe Development

## Table of Contents

- [What Are Enums](#what-are-enums)
- [Enum Basics in Laravel](#enum-basics-in-laravel)
  - [Creating Enums](#creating-enums)
  - [Enum Cases](#enum-cases)
  - [Basic Usage](#basic-usage)
  - [Custom Methods in Enums](#custom-methods-in-enums)
- [Usage in Laravel Applications](#usage-in-laravel-applications)
  - [Hardcoded Values to Enums](#hardcoded-values-to-enums)
  - [Enums in Database Operations](#enums-in-database-operations)
  - [Understanding Laravel's Casting System](#understanding-laravels-casting-system)
  - [Route Model Binding with Enums](#route-model-binding-with-enums)
  - [Validation with Enums](#validation-with-enums)

## What Are Enums

Enums are a relatively new feature in PHP (introduced in PHP 8.1) and naturally found their way into Laravel.

Enums (short for enumerations) are a special data type that allows you to define a set of named constants. Think of them as a contract that restricts values to a predefined list of options.

Consider a typical scenario where you need to represent a user's subscription status. Without enums, you might use string literals scattered throughout your codebase:

```php
// Problematic approach
$status = 'active'; // Could be 'Active', 'ACTIVE', 'activated', etc.
```

This approach is error-prone and lacks consistency. Enums solve this by creating a controlled vocabulary:

```php
// With enums
$status = SubscriptionStatus::Active; // Clear, consistent, and type-safe
```

Enums act as a bridge between the flexibility of variables and the structure of classes—they provide just enough constraint to prevent errors while maintaining simplicity.

## Enum Basics in Laravel

### Creating Enums

Laravel provides an artisan command to generate enums efficiently:

```bash
php artisan make:enum StatusEnum
```

The command will prompt you to select a type: **pure**, **string**, or **integer**. The difference lies in whether the enum cases have backing values:

**Pure Enum** (no backing values):

```php
<?php

namespace App\Enums;

enum Priority
{
    case High;
    case Medium;
    case Low;
}
```

**String-backed Enum**:

```php
<?php

namespace App\Enums;

enum Status: string
{
    case Published = 'published';
    case Draft = 'draft';
    case Archived = 'archived';
}
```

**Pro tip**: For better organization, create your first enum with a namespace:

```bash
php artisan make:enum Enums/Status
```

This creates an `Enums` folder in your `App` directory. Subsequent enums will automatically be placed there.

### Enum Cases

Cases are the heart of any enum—they define the actual values your enum can represent. Each case is a singleton instance of the enum:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
}
```

### Basic Usage

You can access enum cases and their properties in several ways:

```php
// Access the case directly
$status = OrderStatus::Pending;

// Get the name and value
echo $status->name;  // 'Pending'
echo $status->value; // 'pending'

// Get all cases
$allStatuses = OrderStatus::cases(); // Returns array of all cases
```

One significant advantage of enums is **IDE autocomplete support**. Your editor will suggest available cases, reducing typos and improving development speed.

### Custom Methods in Enums

Enums are not just data containers—you can give them behavior. enums can contain methods for additional functionality like labels, descriptions, or associated data:

```php
<?php

namespace App\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Order Pending',
            self::Processing => 'Being Processed',
            self::Shipped => 'In Transit',
            self::Delivered => 'Successfully Delivered',
            self::Cancelled => 'Order Cancelled',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Pending => '#6c757d',
            self::Processing => '#0d6efd',
            self::Shipped => '#fd7e14',
            self::Delivered => '#198754',
            self::Cancelled => '#dc3545',
        };
    }

    public function description(): string
    {
        return match ($this) {
            self::Pending => 'Your order is waiting to be processed',
            self::Processing => 'We are preparing your order',
            self::Shipped => 'Your order is on its way',
            self::Delivered => 'Your order has been delivered',
            self::Cancelled => 'This order has been cancelled',
        };
    }
}
```

## Usage in Laravel Applications

### Hardcoded Values to Enums

Let's see how enums improve a typical controller scenario. Consider this problematic approach:

**Before (with hardcoded values):**

```php
// Controller
class PostController extends Controller
{
    public function index()
    {
        $posts = Post::all();
        $availableStatuses = ['draft', 'published', 'archived']; // Hardcoded

        return response()->json([
            'posts' => $posts,
            'statuses' => $availableStatuses
        ]);
    }
}
```

```html
<script setup>
  import { ref } from "vue";

  const statusOptions = ref([
    { value: "draft", label: "Draft" },
    { value: "published", label: "Published" },
    { value: "archived", label: "Archived" },
  ]); // Duplicated logic
</script>
```

**After (with enums):**

```php
// Controller
class PostController extends Controller
{
    public function index()
    {
        $posts = Post::all();

        return response()->json([
            'posts' => $posts,
            'statuses' => collect(PostStatus::cases())->map(fn($status) => [
                'value' => $status->value,
                'label' => $status->label(),
                'color' => $status->color()
            ])
        ]);
    }
}
```

```html
<script setup>
  import { ref, onMounted } from "vue";
  import axios from "axios";

  const statusOptions = ref([]);

  onMounted(async () => {
    const response = await axios.get("/posts");
    statusOptions.value = response.data.statuses;
  });
</script>
```

### Enums in Database Operations

While Laravel migrations support an `enum` column type:

```php
$table->enum('status', ['draft', 'published', 'archived']);
```

**This approach is not recommended** due to several limitations:

- Difficult to modify enum values without migrations
- Database-specific implementation
- Poor version control handling
- Limited flexibility

Instead, use a string column and leverage Laravel's casting system:

```php
// Migration
$table->string('status');
```

```php
// Model
class Post extends Model
{
    protected function casts(): array
    {
        return [
            'status' => PostStatus::class,
            'tags' => AsEnumCollection::of(TagEnum::class),
            'published_at' => 'datetime',
            'priority_score' => 'float',
        ];
    }
}
```

### Understanding Laravel's Casting System

Casting is Laravel's way of automatically converting database values to specific PHP types when retrieving models, and back to appropriate database formats when saving. This happens transparently:

```php
// When retrieving from database
$post = Post::find(1);
$post->status; // Returns PostStatus enum instance, not string
$post->published_at; // Returns Carbon instance, not string
$post->priority_score; // Returns float, not string

// When saving to database
$post->status = PostStatus::Published; // Automatically converts to 'published'
$post->save();
```

Common cast types include:

- `'datetime'` - Converts timestamps to Carbon instances
- `'float'` - Ensures numeric values are floats
- `'boolean'` - Converts 1/0 to true/false
- `'array'` - JSON encodes/decodes arrays
- `'encrypted'` - Automatically encrypts/decrypts sensitive data

### Route Model Binding with Enums

Laravel's implicit route model binding works seamlessly with enums:

```php
// Route
Route::get('posts/status/{status}', [PostController, 'byStatus']);

// Controller
public function byStatus(PostStatus $status)
{
    // $status is automatically resolved to the enum instance
    $posts = Post::where('status', $status)->get(); // Laravel uses $status->value automatically

    return response()->json([
        'status' => [
            'name' => $status->name,
            'label' => $status->label(),
            'color' => $status->color()
        ],
        'posts' => $posts
    ]);
}
```

### Validation with Enums

Replace verbose validation rules with clean enum validation:

**Before:**

```php
$request->validate([
    'status' => ['required', 'string', 'in:draft,published,archived'],
]);
```

**After:**

```php
use Illuminate\Validation\Rule;

$request->validate([
    'status' => ['required', Rule::enum(PostStatus::class)],
]);
```

The enum validation automatically ensures the value matches one of the defined cases, providing better error messages and maintaining consistency with your enum definition.
