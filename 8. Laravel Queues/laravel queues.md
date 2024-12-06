# Table of Contents

1. [Introduction](#1-introduction)

2. [What is Task Queuing](#2-what-is-task-queuing)

3. [Queues, Jobs and Workers](#3-queues-jobs-and-workers)

4. [Queue Connections and Drivers](#4-queue-connections-and-drivers)
   
   - [a- Supported Queue Drivers/Connections](#a--supported-queue-driversconnections)
   
   - [b- Configuration](#b--configuration)
   
   - [c- Switching Connections](#c--switching-connections)

5. [Jobs](#5-jobs)
   
   - [a- Class Structure](#a--class-structure)
   - [b- Model Serialization](#b--model-serialization)

6. [Dispatching Jobs](#6-dispatching-jobs)
   
   - [a- Conditional Dispatch](#a--conditional-dispatch)
   - [b- Delayed Dispatching](#b--delayed-dispatching)
   - [c- Database Transactions](#c--database-transactions)
   - [d- Queueing Closures](#d--queueing-closures)

7. [Job Chaining](#7-job-chaining)

8. [Named Queues](#8-named-queues)

9. [Queue Worker](#9-queue-worker)
   
   - [a- Works On Production](#a--works-on-production)

10. [Error Handling](#10-error-handling)
    
    - [a- Automatic retry on failure](#a--automatic-retry-on-failure)
    - [b- Custom error handling](#b--custom-error-handling)
    - [c- Retrying failed jobs](#c--retrying-failed-jobs)
    - [d- Ignoring Missing Models](#d--ignoring-missing-models)

11. [Queue Jobs vs. Cron Jobs](#11-queue-jobs-vs-cron-jobs)

12. [Resources](#resources)

# 1. Introduction

While building your web application, you may have some heavy tasks, that take too long to perform during a typical web request.

For example, a database query that touches millions of records, or think about communicating with a third-party service that you have no control over.

In most cases you don't have to let the user wait until the task or all the tasks inside the request to finish completely before you respond to them. In these cases it's better that you send these heavy tasks or slow tasks to the background and respond to the user immediately.

Handling tasks in the background is called asynchronous task execution or task queuing. it's very complicated and requires a lot of work unless you are using laravel.

# 2. What is Task Queuing

Task queuing is a method employed to manage and execute tasks asynchronously in web applications.

It provides a means to decouple time-consuming or resource-intensive operations from the main application flow, allowing the application to respond quickly to user requests.

Instead of executing tasks immediately, they are placed in a queue for later processing by dedicated workers.

This separation of concerns ensures that the application remains responsive, providing a smoother user experience.

# 3. Queues, Jobs and Workers

Queues, jobs and workers are the main concepts and terms in task queuing in Laravel:

- **Jobs**: A job represents a unit of work that needs to be executed asynchronously. Jobs encapsulate the task to be performed and can include data or instructions required for its execution.
- **Queues**: Queues are named channels where jobs are placed for processing. You can configure multiple queues to prioritise and organise tasks based on their importance and resource requirements.
- **Workers**: Workers are processes or threads responsible for executing queued jobs. They continuously monitor the task queue and pick up jobs for execution.

<img src="queues%20illustration.png" title="" alt="" width="652">

# 4. Queue Connections and Drivers

Laravel's queue system provides a flexible configuration that supports multiple queue connections and drivers. A queue connection defines how and where your jobs will be stored and processed.

## a- Supported Queue Drivers/Connections

Laravel offers several built-in queue drivers, each with unique characteristics:

1. **Database Driver**
   - Stores jobs directly in your database
   - Good for small to medium-sized applications
   - Requires minimal additional infrastructure
   - Easy to set up and monitor
   - Performance can degrade with large job volumes
2. **Redis Driver**
   - Uses Redis as a high-performance queue backend
   - Extremely fast and lightweight
   - Supports advanced features like job prioritization
   - Ideal for applications with high-throughput queuing needs
   - Requires Redis server installation
3. **Amazon SQS**
   - Cloud-based queue service by Amazon Web Services
   - Highly scalable and reliable
   - Excellent for distributed systems
   - Pay-per-use pricing model
   - Requires AWS account and credentials
4. **Beanstalkd**
   - Dedicated job queue server
   - Simple and efficient
   - Good for background job processing
   - Less commonly used compared to other drivers
5. **Sync Driver**
   - Processes jobs immediately during the request
   - Useful for testing and local development
   - Does not provide actual asynchronous processing

## b- Configuration

The queue configuration is located in `config/queue.php`. Here's a typical configuration structure:

```php
return [
    'default' => env('QUEUE_CONNECTION', 'sync'),

    'connections' => [
        'sync' => [
            'driver' => 'sync',
        ],

        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => 'default',
            'retry_after' => 90,
        ],
        // Other connection configurations...
    ],
];
```

## c- Switching Connections

You can specify a different connection when dispatching a job:

```php
// Dispatch a job to a specific connection
SendReportJob::dispatch()->onConnection('redis');
```

Or set a default connection in your `.env` file:

```php
QUEUE_CONNECTION=redis
```

**<u>Best Practices</u>**

1. Match your queue driver to your application's scale and performance requirements
2. Consider infrastructure complexity and maintenance
3. Use environment variables for flexible configuration
4. Monitor queue performance and adjust as needed

# 5. Jobs

By default, all of the queueable jobs for your application are stored in the `app/Jobs` directory. If the `app/Jobs` directory doesn't exist, it will be created when you run the `make:job` Artisan command:

```shell
php artisan make:job ProcessPodcast
```

The generated class will implement the `Illuminate\Contracts\Queue\ShouldQueue` interface, indicating to Laravel that the job should be pushed onto the queue to run asynchronously.

## a- Class Structure

Job classes are very simple, normally containing only a `constructor` and a `handle` method that is invoked when the job is processed by the queue.

```php
class SendEmailsJob implements ShouldQueue
{
    use Queueable;

    public function __construct()
    {
        //
    }

    public function handle(): void
    {
        //
    }
}
```

The `handle` method is invoked when the job is processed by the queue. Note that we are able to type-hint dependencies on the `handle` method of the job. The Laravel **<u>service container</u>** automatically injects these dependencies.

## b- Model Serialization

If your queued job accepts an Eloquent model in its constructor, only the identifier for the model will be serialized onto the queue.

When the job is actually handled, the queue system will automatically re-retrieve the full model instance and its loaded relationships from the database.

This approach to model serialization allows for much smaller job payloads to be sent to your queue driver.

Because all loaded Eloquent model relationships also get serialized when a job is queued, the serialized job string can sometimes become quite large. Furthermore, when a job is deserialized and model relationships are re-retrieved from the database, they will be retrieved in their entirety.

To prevent relations from being serialized, you can call the `withoutRelations` method on the model when setting a property value. This method will return an instance of the model without its loaded relationships:

```php
public function __construct(Order $order)
{
    $this->order= $order->withoutRelations();
}
```

# 6. Dispatching Jobs

Dispatching a job in simple works is sending it to the queue, or to the asynchronous execution.

Once you have written your job class, you may dispatch it using the `dispatch` method on the job itself.

The arguments passed to the `dispatch` method will be given to the job's constructor:

You can dispatch it from any part of your application, such as a controller, route, or another job.

```php
class OrdersController extends Controller
{
    public function store(Request $request)
    {
        $order= Order::create(/* ... */);

        // ...

        SendEmailsJob::dispatch($order);

        return redirect('/orders');
    }
}
```

## a- Conditional Dispatch

If you would like to conditionally dispatch a job, you may use the `dispatchIf` and `dispatchUnless` methods:

```php
SendEmailsJob::dispatchIf($accountActive, $order);

SendEmailsJobs::dispatchUnless($accountSuspended, $order);
```

## b- Delayed Dispatching

If you would like to specify that a job should not be immediately available for processing by a queue worker, you may use the `delay` method when dispatching the job.

```php
SendEmailsJob::dispatch($order)->delay(now()->addMinutes(10));
```

## c- Database Transactions

When dispatching a job within a transaction, it is possible that the job will be processed by a worker before the parent transaction has committed. When this happens, any updates you have made to models or database records during the database transaction(s) may not yet be reflected in the database.

Laravel provides several methods of working around this problem. First, you may set the `after_commit` connection option in your queue connection's configuration array:

```php
'database' => [
            'driver' => 'database',
            'connection' => env('DB_QUEUE_CONNECTION'),
            'table' => env('DB_QUEUE_TABLE', 'jobs'),
            'queue' => env('DB_QUEUE', 'default'),
            'retry_after' => (int) env('DB_QUEUE_RETRY_AFTER', 90),
            'after_commit' => true,
],
```

When the `after_commit` option is `true`, Laravel will wait until the open parent database transactions have been committed before actually dispatching the job.

If a transaction is rolled back due to an exception that occurs during the transaction, the jobs that were dispatched during that transaction will be discarded.

The second option is to chain the `afterCommit` method onto your dispatch operation:

```php
ProcessPodcast::dispatch($podcast)->afterCommit();
```

## d- Queueing Closures

Instead of dispatching a job class to the queue, you may also dispatch a closure. This is great for quick, simple tasks that need to be executed outside of the current request cycle. 

```php
$podcast = App\Podcast::find(1);

dispatch(function () use ($podcast) {
    $podcast->publish();
});
```

Using the `catch` method, you may provide a closure that should be executed if the queued closure fails to complete successfully

```php
use Throwable;

dispatch(function () use ($podcast) {
    $podcast->publish();
})->catch(function (Throwable $e) {
    // This job has failed...
});
```

# 7. Job Chaining

Job chaining is the process of linking multiple jobs together to form a chain of tasks. Each job in the chain is executed sequentially, meaning that the next job in the chain will only start once the previous one has completed successfully.

Consider a user registration workflow where you need to perform multiple tasks in a specific sequence:

1. Create a new user record in the database
2. Send a welcome email to the user
3. Log the registration activity
4. Notify an admin about the new registration

To execute a queued job chain, you may use the `chain` method provided by the `Bus` facade.

```php
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new CreateUserJob,
    new SendWelcomeEmailJob,
    new LogRegistrationActivityJob,
    new NotifyAdminJob
])->dispatch();
```

In addition to chaining job class instances, you may also chain closures:

```php
Bus::chain([
    new MakeOrderJob,
    new SendConfirmationEmail,
    function () {
        Order::update(/* ... */);
    },
])->dispatch();
```

When chaining jobs, you may use the `catch` method to specify a closure that should be invoked if a job within the chain fails. The given callback will receive the `Throwable` instance that caused the job failure:

```php
use Illuminate\Support\Facades\Bus;
use Throwable;

Bus::chain([
    new FirstJob,
    new SecondJob,
    new LastJob,
])->catch(function (Throwable $e) {
    // A job within the chain has failed...
})->dispatch();
```

# 8. Named Queues

You can push jobs to particular queues. To specify the queue, use the `onQueue` method when dispatching the job:

```php
 SendEmailsJob::dispatch($order)->onQueue('emails');
```

Alternatively, you may specify the job's queue by calling the `onQueue` method within the job's constructor:

```php
public function __construct()
    {
        $this->onQueue('processing');
    }
```

Naming queues helps on  optimization, by separating high-priority and low-priority tasks.

# 9. Queue Worker

Laravel includes an Artisan command that will start a queue worker and process new jobs as they are pushed onto the queue. You may run the worker using the `queue:work` Artisan command. Note that once the `queue:work` command has started, it will continue to run until it is manually stopped or you close your terminal:

```shell
php artisan queue:work
```

workers execute jobs in a **FIFO** **(first in first out)** manner, that means all the older jobs will be executed first before the jobs that are more recent.

By default, the `queue:work` command only processes jobs for the default queue on a given connection.

However, you may customize your queue worker even further by only processing particular queues for a given connection. For example, if all of your emails are processed in an `emails` queue on your `redis` queue connection, you may issue the following command to start a worker that only processes that queue:

```shell
php artisan queue:work redis --queue=emails
```

## a- Works On Production

**On the Server**: The worker must run continuously to handle jobs reliably. This is achieved using a **process supervisor**, like `Supervisor` on Linux.

<u>**Steps to Manage Workers on Your Server**</u>

1. **Install Supervisor**:
   
   - Install it on your server (`sudo apt install supervisor`).
   - It ensures the queue worker runs continuously and restarts if it crashes.

2. **Configure Supervisor for Each Project**:
   Create a configuration file for each Laravel project in `/etc/supervisor/conf.d/`. For example:
   
   ```ini
   [program:project1-worker]
   process_name=%(program_name)s_%(process_num)02d
   command=php /path/to/project1/artisan queue:work --tries=3
   autostart=true
   autorestart=true
   user=your_server_user
   numprocs=1
   redirect_stderr=true
   stdout_logfile=/path/to/project1/storage/logs/queue.log
   ```

    Replace `/path/to/project1` with the actual path to your Laravel project.

3. **Reload and Start Supervisor**:
   After adding the configuration file:
   
   ```shell
   sudo supervisorctl reread
   sudo supervisorctl update
   sudo supervisorctl start project1-worker
   ```

**<u>Managing Multiple Projects</u>**

- **Separate Configurations**: Each project gets its own Supervisor configuration file with a unique `program` name.
- **Separate Queue Connections**: You can set up different queue connections for each project (e.g., Redis database 1 for Project A, Redis database 2 for Project B).

**<u>Alternatives to Supervisor</u>**

- **Systemd**: A modern alternative to Supervisor, where you define a `.service` file for each project.
- **Docker**: If you use Docker, you can run queue workers in separate containers for each project.

Since queue workers are long-lived processes, they will not notice changes to your code without being restarted. So, the simplest way to deploy an application using queue workers is to restart the workers during your deployment process. You may gracefully restart all of the workers by issuing the `queue:restart` command:

```php
php artisan queue:restart
```

# 10. Error handling

Error handling is a critical aspect of working with queued jobs in Laravel. When jobs are executed asynchronously, it's essential to have robust error-handling strategies in place to ensure the reliability of your application.

## a- Automatic retry on failure

Laravel allows you to specify how many times a job should be retried if it fails. This is useful for handling transient errors, such as network timeouts or temporary resource unavailability. You can configure the number of retries by adding the `$tries` property to your job class, such as in this example:

```php
public $tries = 3;
```

ALso when running a queue worker process, you may specify the maximum number of times a job should be attempted using the `--tries` switch on the `queue:work` command.

```shell
php artisan queue:work redis --tries=3
```

## b- Custom error handling

You can implement custom error-handling logic within your job classes by defining the `failed()` method. This method is called when a job fails its maximum number of retry attempts. Here, you can log errors, send notifications, or take other appropriate actions.

```php
public function failed(Exception $exception) {
    // Log the error or send a notification
}
```

## c- Retrying failed jobs

To view all of the failed jobs that have been inserted into your `failed_jobs` database table, you may use the `queue:failed` Artisan command:

```shell
php artisan queue:failed
```

The `queue:failed` command will list the job ID, connection, queue, failure time, and other information about the job. The job ID may be used to retry the failed job. For instance, to retry a failed job that has an ID of `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece`, issue the following command:

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```

If necessary, you may pass multiple IDs to the command:

```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d
```

You may also retry all of the failed jobs for a particular queue:

```shell
php artisan queue:retry --queue=name
```

To retry all of your failed jobs, execute the `queue:retry` command and pass `all` as the ID:

```shell
php artisan queue:retry all
```

If you would like to delete a failed job, you may use the `queue:forget` command:

```shell
php artisan queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d
```

To delete all of your failed jobs from the `failed_jobs` table, you may use the `queue:flush` command:

```shell
php artisan queue:flush
```

## d- Ignoring Missing Models

When injecting an Eloquent model into a job, the model is automatically serialized before being placed on the queue and re-retrieved from the database when the job is processed. However, if the model has been deleted while the job was waiting to be processed by a worker, your job may fail with a `ModelNotFoundException`.

For convenience, you may choose to automatically delete jobs with missing models by setting your job's `deleteWhenMissingModels` property to `true`. When this property is set to `true`, Laravel will quietly discard the job without raising an exception:

```php
public $deleteWhenMissingModels = true;
```

# 11. Queue Jobs vs. Cron Jobs

While both queue jobs and cron jobs involve task execution, they serve different purposes in software development.

## Queue Jobs

Queue jobs are asynchronous tasks executed within a web application. They are programmatically triggered to handle time-consuming operations without blocking the main application flow.

**Characteristics**:

- Executed within the application context
- Triggered by specific events or actions
- Processed by dedicated queue workers
- Ideal for tasks like sending emails, processing uploads, or generating reports

**Example Scenarios**:

- Sending welcome emails after user registration
- Generating PDF reports
- Processing large file uploads
- Synchronizing data with external services

## Cron Jobs

Cron jobs are system-level scheduled tasks that run at predetermined intervals, independent of web application logic.

**Characteristics**:

- Scheduled at system level
- Executed by the server's scheduling mechanism
- Run at fixed time intervals
- Typically used for system maintenance and background operations

**Example Scenarios**:

- Daily database backups
- Cleaning up temporary files
- Generating system reports
- Performing routine system maintenance

## Laravel's Task Scheduler

Laravel provides a task scheduler that bridges the gap between traditional cron jobs and application-level task management, offering a more integrated approach to scheduling tasks within the framework.

**Key Difference**: Queue jobs handle asynchronous application-specific tasks, while cron jobs manage system-wide scheduled operations.

# Resources

1. **Great Video From Mohamed Said About The Queues**  
    [The Queue Component in Laravel - YouTube](https://www.youtube.com/watch?v=m-hNL87-lFo&t=30s)

2. **An Article From Twilio About Queues in Laravel**  
   [How to Use Queueing in Laravel: Setup and Manage | Twilio](https://www.twilio.com/en-us/blog/queueing-in-laravel)

3. **And Of Course The Great Official Docs**
   
   [Queues - Laravel 11.x ](https://laravel.com/docs/11.x/queues#dealing-with-failed-jobs)
