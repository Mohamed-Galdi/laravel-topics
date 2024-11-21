# Table of Contents
1. [API  Versioning](#1-api-versioning)
   - [a. Controller Versioning](#a-controller-versioning)
   - [b. Route Versioning](#b-route-versioning)
2. [Resources and Collections](#2-resources-and-collections)
    - [a. Resources](#a-resources)
    - [b. Collections](#b-collections)
    - [c. Pagination](#c-pagination)
    - [d. Conditional Attributes](#d-conditional-attributes)
    - [e. Relationships](#e-relationships)
3. [Filters](#3-filters)
    - [a. Filter Class](#a-filter-class)
    - [b. Base Filter Class](#b-base-filter-class)
4. [Form Request](#4-form-request)
5. [API Authentication](#5-api-authentication)
6. [API Documentation](#6-api-documentation)

# 1. API Versioning

An API is essentially a software product we release, maintain, and enhance over time. Developers use APIs in their applications (web or mobile), so any changes to the API could potentially break their apps.

To prevent disruptions, **versioning** is crucial. It allows you to release new features without affecting existing API consumers. In Laravel, you should version components that directly affect the API users, like controllers and routes, while stable elements like models usually don’t require versioning.

## a. Controller Versioning

Organize versioned controllers by creating folders for each API version inside the `App\Http\Controllers\Api` directory.

![](api%20controller%20versioning%20folders.png)

**Note:** The base `Controller` class located in `App\Http\Controllers` is imported automatically for controllers in the main folder. However, for subfolders, you need to import it manually.

```php
namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller; // Import base Controller class

class ProductsController extends Controller
{
    // Controller logic here
}
```

## b. Route Versioning

API routes are defined in the `routes/api.php` file. In Laravel 11, this file isn’t available by default but can be created with the following Artisan command:

```shell
php artisan api:install
```

Routes defined in `api.php` are automatically prefixed with `/api`. You can customize this prefix in the `RouteServiceProvider`.

To version routes, use the `group()` method to group routes by version. Here's an example:

```php
Route::group([
    'prefix' => 'v1',
    'namespace' => 'App\Http\Controllers\Api\V1'
], function () {
    Route::apiResource('products', ProductsController::class);
});
```

**Explanation:**

- **`prefix`**: Adds `/v1` to the beginning of all routes in this group (e.g., `/api/v1/products`).
- **`namespace`**: Specifies the namespace where the controllers for this version are located.

### Additional Notes:

1. Keep your directory structure consistent for clarity and scalability.
2. Avoid hardcoding namespaces in routes if you're using Laravel 8+ because namespace resolution for controllers is automatic by default. Use the full controller path in `Route::apiResource()` if you disable automatic namespace detection.

# 2. Resources and Collections

When building APIs, transforming Eloquent models into JSON responses is often necessary. Laravel's resource classes provide a flexible and expressive way to define how your models and their collections are transformed into JSON.

## a. Resources

Resources serve as a transformation layer between your Eloquent models and the JSON responses sent to API consumers. They allow you to define which attributes to expose, customize naming conventions, and include specific relationships.

You can create a resource using the following Artisan command: 

```shell
php artisan make:resource V1/ProductResource
```

The generated resource will be located at: `app/Http/Resources/V1/ProductResource.php`

A resource class extends `JsonResource` and defines a `toArray()` method to specify the data returned in the API response.

Example Resource Class:

```php
namespace App\Http\Resources\V1;

use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'price' => $this->price,
        ];
    }
}
```

Controller Example:

```php
namespace App\Http\Controllers\Api\V1;

use App\Models\Product;
use App\Http\Resources\V1\ProductResource;

class ProductsController extends Controller
{
    public function show(Product $product)
    {
        return new ProductResource($product);
    }
}

```

API Response:

```php
{
    "data": {
        "id": 1,
        "name": "Sample Product",
        "description": "A sample product description.",
        "price": 100
    }
}

```

By default, resource responses are wrapped in a `data` object. This behavior is a convention that promotes consistency and is generally recommended, though you can customize it if needed.

**<u>Naming Conventions:</u>**

Laravel's default database naming uses snake_case, while JSON APIs typically follow camelCase. You can use resources to handle this transformation:

```php
'name' => $this->product_name,
```

## b. Collections

Collections transform multiple models into JSON responses and can include additional metadata like links or pagination details. While a resource represents a single model, a collection represents multiple models.

You can generate a collection resource with the `--collection` flag or by including "Collection" in the resource name:

```shell
php artisan make:resource Product --collection
php artisan make:resource ProductCollection
```

The collection class extends `Illuminate\Http\Resources\Json\ResourceCollection`.

Example Collection Class:

```php
namespace App\Http\Resources\V1;

use Illuminate\Http\Resources\Json\ResourceCollection;

class ProductCollection extends ResourceCollection
{
    public function toArray($request)
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => 'link-value',
            ],
        ];
    }
}

```

Returning a Collection:

```php
use App\Http\Resources\V1\ProductCollection;
use App\Models\Product;

Route::get('/products', function () {
    return new ProductCollection(Product::all());
});

```

If you don’t need a separate collection class, you can use the `collection()` method directly on the resource:

```php
use App\Http\Resources\V1\ProductResource;
use App\Models\Product;

Route::get('/products', function () {
    return ProductResource::collection(Product::all());
});

```

## c. Pagination

To include pagination in your API responses, pass a paginator instance to the `collection()` method or a custom resource collection:

```php
use App\Http\Resources\V1\ProductCollection;
use App\Models\Product;

Route::get('/products', function () {
    return new ProductCollection(Product::paginate());
});
```

Paginated responses always contain `meta` and `links` keys with information about the paginator's state:

```json
{
    "data": [
        {
            "id": 1,
            "name": "Product 1",
            "price": 100
        },
        {
            "id": 2,
            "name": "Product 2",
            "price": 200
        }
    ],
    "links": {
        "first": "http://example.com/products?page=1",
        "last": "http://example.com/products?page=3",
        "prev": null,
        "next": "http://example.com/products?page=2"
    },
    "meta": {
        "current_page": 1,
        "last_page": 3,
        "total": 30,
        "per_page": 10
    }
}
```

## d. Conditional Attributes

You can conditionally include attributes in the resource response using methods like `when()`, `whenHas()`, or `whenNotNull()`.

**Examples:**

- **Using `when`:** 
  
  ```php
  'secret' => $this->when($request->user()->isAdmin(), 'secret-value'),
  ```

- **Using `whenHas`:**
  
  ```php
  'name' => $this->whenHas('name'),
  ```
- **Using `whenNotNull`:**
  
  ```php
  'name' => $this->whenNotNull($this->name),
  ```

## e. Relationships

You can include relationships using the `whenLoaded()` method. This ensures relationships are only included if they’ve already been loaded.

```php
use App\Http\Resources\V1\ProductResource;

public function toArray($request)
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'category' => new CategoryResource($this->whenLoaded('category')),
    ];
}

```

**Example Controller:**

```php
use App\Models\Product;
use App\Http\Resources\V1\ProductResource;

$product = Product::with('category')->find(1);
return new ProductResource($product);

```

API Response:

```php
{
    "data": {
        "id": 1,
        "name": "Sample Product",
        "category": {
            "id": 1,
            "name": "Electronics"
        }
    }
}
```

# 3. Filters

When building an API, providing users with the ability to filter data is essential. Filters allow users to refine the data they retrieve based on specific conditions. Filters are implemented using query parameters, and Laravel makes this process highly customizable.

**<u>Filtering with Query Parameters</u>**:

Filters depend on query parameters sent by the client. For example:

1. **Basic Filtering**  
   `url/products?price=20`  
   
   - In this case, the query parameter `price` directly represents the value `20`.
   
   - Laravel interprets it as a simple key-value pair, with `price` as the key and `20` as the value.

1. **Advanced Filtering with Operators**  
   `url/products?price[gt]=20`  
   
   - Here, the query parameter `price` is an array (`gt` is a key within that array).
   - Laravel parses this into a structured object: `['gt' => '20']`.
   - This allows you to specify a comparison operator (`gt`, `lt`, etc.) along with the value.


![](query%20operators.png)

## a. Filter Class

Filters can quickly grow complex when you have multiple parameters and conditions. Hardcoding them into your controllers makes your code harder to maintain and less reusable. Filter classes provide a structured way to handle these operations.

Here’s an example of a custom filter class:

```php
namespace App\Filters\V1;

use Illuminate\Http\Request;

class ProductFilter {
    // Define allowed parameters and operators for filtering
    protected $allowedParams = [
        'name' => ['eq'],
        'price' => ['eq', 'gt', 'lt'],
    ];

    // Map query parameters to database column names
    protected $columnMap = [
        'productionDate' => 'prod_date',
    ];

    // Map custom operators to Eloquent operators
    protected $operatorMap = [
        'eq' => '=',
        'lt' => '<',
        'gt' => '>',
    ];

    /**
     * Transform the query parameters into filterable Eloquent conditions.
     */
    public function transform(Request $request): array {
        $queries = [];

        foreach ($this->allowedParams as $param => $operators) {
            $query = $request->query($param);

            // Skip if the parameter is not present in the request
            if (!isset($query)) {
                continue;
            }

            // Map the parameter to the corresponding column name
            $column = $this->columnMap[$param] ?? $param;

            // Check for allowed operators
            foreach ($operators as $operator) {
                if (isset($query[$operator])) {
                    $queries[] = [$column, $this->operatorMap[$operator], $query[$operator]];
                }
            }
        }

        return $queries;
    }
}

```

Once the filter class is defined, you can use it in your controller to dynamically generate queries:

```php
namespace App\Http\Controllers;

use App\Filters\V1\ProductFilter;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller {
    public function index(Request $request) {
        $filter = new ProductFilter();
        $queries = $filter->transform($request);

        if (count($queries) === 0) {
            return response()->json(Product::paginate(10));
        }

        // Apply the generated queries to the Eloquent model
        $products = Product::where($queries)->paginate(10);

        // Append query parameters to pagination links
        return response()->json($products->appends($request->query()));
    }
}

```

## b. Base Filter Class

If your API contains multiple models that require filtering, you can create a **base filter class** to handle shared logic, and extend it for specific filters.

```php
namespace App\Filters;

use Illuminate\Http\Request;

class ApiFilter {
    protected $allowedParams = [];
    protected $columnMap = [];
    protected $operatorMap = [
        'eq' => '=',
        'lt' => '<',
        'gt' => '>',
    ];

    public function transform(Request $request): array {
        $queries = [];

        foreach ($this->allowedParams as $param => $operators) {
            $query = $request->query($param);

            if (!isset($query)) {
                continue;
            }

            $column = $this->columnMap[$param] ?? $param;

            foreach ($operators as $operator) {
                if (isset($query[$operator])) {
                    $queries[] = [$column, $this->operatorMap[$operator], $query[$operator]];
                }
            }
        }

        return $queries;
    }
}

```

Extending the Base Filter Class:

```php
namespace App\Filters\V1;

use App\Filters\ApiFilter;

class ProductFilter extends ApiFilter {
    protected $allowedParams = [
        'name' => ['eq'],
        'price' => ['eq', 'gt', 'lt'],
    ];

    protected $columnMap = [
        'productionDate' => 'prod_date',
    ];
}

```

**Controller Usage:**

```php
use App\Filters\V1\ProductFilter;
use App\Models\Product;

public function index(Request $request) {
    $filter = new ProductFilter();
    $queries = $filter->transform($request);

    return response()->json(Product::where($queries)->paginate(10)->appends($request->query()));
}

```

**Note:** While this filtering approach is effective and secure, it can be further enhanced to support additional features like dynamic query building, validation for filter values, handling `OR` conditions, partial matching (e.g., `LIKE` queries), sorting (e.g., `?sort=price:asc`), eager loading related data (e.g., `?include=category,tags`), and caching for frequently used filters. These improvements can make the filtering system more flexible and scalable for complex API requirements.

# 4. Form Request

In Laravel, validation is often handled directly within the controller. While this is functional, it goes against the **Single Responsibility Principle**, which suggests that a controller should focus solely on receiving requests and returning responses. Validation logic should be delegated elsewhere for cleaner and more maintainable code.

Laravel offers **Form Requests**, which are custom request classes designed specifically for handling validation and authorization logic.

To generate a Form Request class, use the following Artisan command:

```shell
php artisan make:request StoreProductRequest
```

This creates a file in the `app/Http/Requests` directory. Form Requests are versatile and can be used for both API and web applications.

**<u>Key Methods in Form Requests</u>**

1. **`authorize()`**: This method determines whether the authenticated user has permission to perform the requested action. Returning `true` grants access, while `false` denies it.
   
   Example:
   
   ```php
   public function authorize()
   {
       $comment = Comment::find($this->route('comment'));
       return $comment && $this->user()->can('update', $comment);
   }
   ```

2. **`rules()`**: This method defines the validation rules for incoming request data.
   
   Example:
   
   ```php
   public function rules(): array
   {
       return [
           'title' => 'required|unique:posts|max:255',
           'body' => 'required',
       ];
   }
   ```

Since Form Requests extend Laravel's base request class, you can access all the request methods, such as `$this->user()` or `$this->route()`.

**<u>Using Form Requests in Controllers</u>**

To apply the Form Request, simply type-hint it in your controller method:

```php
public function store(StoreProductRequest $request): RedirectResponse
{
    // The request is already validated and authorized.
    $validated = $request->validated(); // Retrieves all validated data.
    
    // Optionally retrieve specific portions of validated data:
    $only = $request->safe()->only(['name', 'email']);
    $except = $request->safe()->except(['password']);

    // Process the validated data (e.g., store in the database).

    return redirect('/posts');
}
```

**<u>Handling Validation Errors</u>**

If validation fails:

- For **web requests**, the user is redirected back to the previous page, and validation errors are flashed to the session.
- For **API requests**, an HTTP 422 response with a JSON representation of the errors is returned.  
  Ensure the `Accept` header is set to `application/json` when testing via tools like Postman.

**<u>Customizing Error Messages</u>**

You can customize validation error messages by overriding the `messages()` method in your Form Request:

```php
public function messages(): array
{
    return [
        'title.required' => 'A title is required.',
        'body.required' => 'The body field is required.',
    ];
}
```

# 5. API Authentication

For securing your API, **Sanctum** is the recommended package. It supports both **session-based authentication** (ideal for SPAs) and **token-based authentication** (suitable for mobile apps and APIs).

To get started quickly, you can use a starter kit like **Laravel Breeze**, which provides an easy setup for authentication.

For a detailed guide on implementing authentication and authorization in Laravel APIs, please refer to my dedicated documentation [here](https://github.com/Mohamed-Galdi/Laravel-Topics/blob/main/1.%20Laravel%20Authentication/Laravel%20auth.md).

# 6. API Documentation

#### a. Why Documenting Your API is Important

API documentation is a crucial part of API development. It serves as a guide for developers who consume your API, helping them understand:

- **How to interact with the API**: Endpoints, request methods, and parameters.
- **Expected responses**: Data formats and potential errors.
- **Business logic**: Any rules or conditions for accessing resources.

Well-documented APIs reduce support requests, improve adoption, and foster better collaboration between frontend and backend developers.

#### b. How to Create API Documentation

Documenting an API involves defining:

1. **Endpoints**: List all routes with HTTP methods (GET, POST, PUT, DELETE, etc.).
2. **Request Parameters**: Specify required and optional parameters, including their data types.
3. **Response Structure**: Show sample responses (success and failure).
4. **Authentication Requirements**: Explain how to authenticate (e.g., tokens, headers).
5. **Error Handling**: List common error codes and messages.
6. **Versioning**: Clarify the version of the API being documented.

#### c. Tools for API Documentation

Several tools can simplify the documentation process:

1. **Swagger/OpenAPI**: A widely used tool that allows you to write documentation in YAML/JSON. It generates an interactive UI for developers to test endpoints.
2. **Postman**: In addition to testing APIs, it can auto-generate API documentation.
3. **Laravel API Documentation Tools**:
   - **Scribe**: A Laravel-specific package for generating API docs automatically from your routes and controllers.
   - **L5-Swagger**: Integrates Swagger into Laravel projects.
4. **API Blueprint**: A markdown-based language for writing API docs.

#### d. Documenting APIs in Laravel with Scribe

For Laravel, **Scribe** is a powerful package for generating API documentation. You can set it up by running:

```shell
composer require knuckleswtf/scribe
php artisan scribe:install
```

It reads your routes and controller methods to generate documentation, including:

- Endpoint descriptions.
- Request and response examples.
- Authentication and error details.

Once installed, run the command to generate documentation:

```shell
php artisan scribe:generate
```

The generated documentation will be accessible via a web interface or a JSON file, making it easy to share with your team or external developers.
