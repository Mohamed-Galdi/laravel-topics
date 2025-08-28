# Monitoring and Load Testing

- [1. What is Load Testing?](#1-what-is-load-testing)
- [2. How to Monitor Server Performance: htop and Beszel Setup Guide](#2-how-to-monitor-server-performance-htop-and-beszel-setup-guide)
  - [a. htop](#a-htop)
  - [b. Beszel](#b-beszel)
- [3. Load Testing Tools and Methods](#3-load-testing-tools-and-methods)
  - [a. Apache Bench (ab)](#a-apache-bench-ab)
  - [b. k6: Load Testing with JavaScript](#b-k6-load-testing-with-javascript)
  - [c. Loader.io](#c-loaderio)
- [4. Load Testing Best Practices](#4-load-testing-best-practices)

## 1. What is Load Testing?

Load testing (aka Stress testing) is the process of deliberately pushing your server beyond its normal operating capacity to identify performance bottlenecks, system limitations, and potential failure points.

During a load test, you simulate high traffic loads, heavy database queries, or intensive computational tasks to observe how your server responds. This might involve sending thousands of simultaneous HTTP requests to your web application, filling up disk space rapidly, or consuming maximum CPU and memory resources.

The goal is to understand your system's breaking point in a controlled environment rather than discovering it when real users are affected.

These tests help you make informed decisions about scaling, optimization, and resource allocation before problems occur in production.

## 2. How to Monitor Server Performance: htop and Beszel Setup Guide

Without proper monitoring, load testing becomes a blind exercise where you might overload your server but have no meaningful data about what actually happened or which components failed first.

Monitoring during load tests provides the essential visibility needed to understand system behavior under load. When you push your server to its limits, you need real-time data about CPU utilization, memory consumption, disk I/O operations, network throughput, and application-specific metrics.

### a. htop

For immediate, real-time system monitoring, `htop` is an indispensable command-line tool that provides a comprehensive view of your server's current state. Unlike the basic `top` command, `htop` offers an intuitive, color-coded interface that displays running processes, CPU usage per core, memory consumption, and system load averages in an easily digestible format.

To install htop on Ubuntu/Debian systems:

```bash
sudo apt update
sudo apt install htop
```

Launch htop by simply typing:

```bash
htop
```

The htop interface shows color-coded CPU usage bars (green for user processes, red for system processes), memory and swap usage, and a sortable list of running processes with their resource consumption.

![htop output example](<htop output example.webp>)

During stress testing, htop becomes your real-time dashboard for observing how individual processes consume system resources and identifying which components become bottlenecks first.

While htop is excellent for real-time observation, it has limitations for long-term analysis and historical data retention. You can't easily track performance trends over time or set up automated alerts when thresholds are exceeded. This is where dedicated monitoring solutions become essential.

### b. Beszel

Beszel is a lightweight, self-hosted server monitoring solution designed specifically for developers and system administrators who need detailed performance insights without the complexity of enterprise monitoring platforms.

Unlike htop's real-time snapshots, Beszel continuously collects, stores, and visualizes performance metrics, providing historical data essential for understanding performance patterns and conducting thorough stress test analysis.

##### Beszel architecture: Hub / Agent

The Beszel architecture consists of two main components: the **Hub** (web dashboard) and **Agent** (data collector).

The **Agent** runs on your server, continuously gathering metrics about CPU usage, memory consumption, disk I/O, network activity, and system load.

This data is transmitted to the **Hub**, which provides a web-based dashboard with charts, graphs, and historical trend analysis.

This separation allows you to monitor multiple servers from a single dashboard while maintaining lightweight resource usage on the monitored systems.

##### Step 1: Configure Your Monitoring Subdomain

Begin by creating a dedicated subdomain for your monitoring dashboard. In your Cloudflare dashboard, navigate to the DNS management section for your domain. Create a new A record with the following configuration:

- **Type**: A
- **Name**: monitor (this creates monitor.yourdomain.com)
- **IPv4 Address**: Your Hetzner server's IP address
- **Proxy Status**: DNS only (gray cloud)

The subdomain approach keeps your monitoring interface separate from your main applications while maintaining easy access through your existing domain infrastructure.

##### Step 2: Install the Beszel Hub

Connect to your server via SSH and install the Beszel Hub using the automated installation script:

```bash
curl -sL https://get.beszel.dev/hub -o /tmp/install-hub.sh
```

```bash
chmod +x /tmp/install-hub.sh
```

```bash
/tmp/install-hub.sh
```

This script downloads the latest Beszel Hub binary and installs it in `/opt/beszel/`. The Hub will run on port 8090 by default and serve as the central dashboard for all your monitoring data.

##### Step 3: Configure Nginx Reverse Proxy

Configure Nginx to serve your Beszel dashboard through the monitoring subdomain. Create a new Nginx server block:

```bash
sudo nano /etc/nginx/sites-available/monitor.yourdomain.com
```

Add the following configuration, replacing `monitor.yourdomain.com` with your actual subdomain:

```nginx
server {
    listen 80;
    server_name monitor.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the new site configuration:

```bash
sudo ln -s /etc/nginx/sites-available/monitor.yourdomain.com /etc/nginx/sites-enabled/
```

Verify your Nginx configuration is valid:

```bash
sudo nginx -t
```

Reload Nginx to apply the changes:

```bash
sudo systemctl reload nginx
```

Obtain and configure an SSL certificate for your monitoring subdomain (assuming you already have certbot installed):

```bash
sudo certbot --nginx -d monitor.yourdomain.com
```

Certbot will automatically modify your Nginx configuration to include SSL settings and redirect HTTP traffic to HTTPS.

##### Step 5: Access Hub and Retrieve Public Key

Navigate to your monitoring subdomain in a web browser: `monitor.yourdomain.com`. You'll first encounter a registration form where you need to create your admin account—fill in your desired username and password to access the dashboard.

After completing registration, you'll be redirected to the main Beszel dashboard interface. Look for the "Add System" or "Systems" section in the dashboard, which will display a public key string needed for connecting your server's agent to the hub.

![beszel public key](<beszel public key.webp>)

Copy this public key and keep it readily available—you'll need it in the next step for agent installation. This key establishes the secure connection between your server's monitoring agent and the web dashboard.

##### Step 6: Install and Configure the Beszel Agent

The agent collects performance data from your server and transmits it to the Hub. Download and install the agent:

```bash
curl -sL https://get.beszel.dev -o /tmp/install-agent.sh
```

```bash
chmod +x /tmp/install-agent.sh
```

```bash
sudo /tmp/install-agent.sh
```

During installation, you'll be prompted to:

- Enter the public key you copied from the Hub dashboard
- Choose whether to enable automatic daily updates (recommended: yes)

The script creates a dedicated system user for the agent and configures it as a systemd service for automatic startup.

##### Step 7: Create Proper Hub Service Configuration

Although the Hub is running, ensure it has a proper systemd service for automatic restart after reboots. First, locate the Beszel binary:

```bash
find /usr/local/bin /opt -name "beszel" 2>/dev/null
```

Create a systemd service file using the correct path (typically `/opt/beszel/beszel`):

```bash
sudo nano /etc/systemd/system/beszel.service
```

Add this service configuration:

```ini
[Unit]
Description=Beszel Hub
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=3
User=root
WorkingDirectory=/opt/beszel
ExecStart=/opt/beszel/beszel serve --http "0.0.0.0:8090"

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable beszel.service
```

```bash
sudo systemctl start beszel.service
```

Your Beszel installation now provides continuous monitoring with historical data retention, real-time alerts, and the detailed metrics necessary for effective stress testing analysis. The dashboard displays CPU usage, memory consumption, disk I/O, network activity, and system load in easy-to-understand charts that update in real-time.

![beszel metrics dashboard](<beszel metrics dashboard.webp>)

## 3. Load Testing Tools and Methods

Once your monitoring infrastructure is in place, the next step is selecting the appropriate load testing tools for your specific requirements. The load testing ecosystem offers numerous options, each designed for different scenarios, technical skill levels, and testing complexity.

We’ll explore three accessible tools that cover most load testing needs: **Apache Bench** for quick, command-line tests, **Loader.io** for easy cloud-based testing without setup, and **k6** for advanced, scriptable scenarios.

### a. Apache Bench (ab)

Apache Bench is one of the simplest load testing tools, suitable for quick performance checks or for learning the basics of load testing.

On Windows, the recommended way to install Apache Bench is through **WSL (Windows Subsystem for Linux)** with Ubuntu. This provides the full version with HTTPS support.

> **Note:** Apache Bench is also included with XAMPP on Windows under `C:\xampp\apache\bin\ab.exe`. However, that build does not support HTTPS, which limits its usefulness for testing modern sites. For that reason, the WSL approach is generally preferred.

First, install it on Ubuntu inside WSL:

```bash
sudo apt update
sudo apt install apache2-utils -y
```

Verify the installation:

```bash
ab -V
```

Once installed, you can run a basic test:

```bash
ab -n 100 -c 10 https://example.com/
```

> **Note:** When running Apache Bench from WSL against a remote site, the results depend not only on the target server but also on your own machine’s resources and internet connection speed and latency.

Apache Bench uses two primary parameters that control the intensity and pattern of your load test:

**-n (Number of requests)**: This parameter specifies the total number of HTTP requests Apache Bench will send to your target server. For example, `-n 1000` means AB will send exactly 1000 requests before completing the test. This determines the overall scope of your test—more requests provide better statistical accuracy but take longer to complete.

**-c (Concurrency level)**: This parameter defines how many requests are sent simultaneously. For example, `-c 10` means Apache Bench maintains 10 concurrent connections to the server. Higher concurrency simulates more users accessing your site at the same time, creating more realistic load conditions.

The relationship between these parameters determines your test pattern. With `-n 1000 -c 10`, Apache Bench sends 10 concurrent requests, waits for responses, then sends the next batch of 10, continuing until all 1000 requests are complete.

##### Reading and Analyzing the Results

After running a test with Apache Bench, you get output that shows server performance metrics. Here is an example of a simplified, realistic result:

```text
Benchmarking example.com (be patient).....done

Server Software:        nginx
Server Hostname:        example.com
Server Port:            443
SSL/TLS Protocol:       TLSv1.3

Document Path:          /
Document Length:        4520 bytes

Concurrency Level:      10
Time taken for tests:   0.85 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      500000 bytes
HTML transferred:       452000 bytes
Requests per second:    118 [#/sec] (mean)
Time per request:       85 [ms] (mean)
Time per request:       8.5 [ms] (mean, across all concurrent requests)
Transfer rate:          580 Kbytes/sec received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2    1.0      2       5
Processing:     10   65   20.0     60     120
Waiting:         8   50   15.0     48     100
Total:          12   67   21.0     62     123

Percentage of the requests served within a certain time (ms)
  50%     62
  66%     70
  75%     80
  80%     85
  90%    100
  95%    110
  98%    120
  99%    123
 100%    123 (longest request)
```

**How to interpret these values:**

- **Requests per second (RPS):** Measures how many requests the server handled per second. Higher is better.

- **Time per request:** Average time for a request to complete. Can be reported per request or across concurrent requests.

- **Failed requests:** Any requests that didn’t complete successfully. Ideally zero.

- **Connection times:** Breaks down how long it took to connect, process, and wait for responses.

- **Percentiles:** Show the distribution of request times. For example, 95% of requests completed within 110 ms.

This output helps you **identify server responsiveness, potential bottlenecks, and overall capacity**.

### b. k6: Load Testing with JavaScript

While Apache Bench is simple and useful for quick tests, **k6** is a modern, scriptable load testing tool designed for more complex scenarios and realistic traffic simulation. It allows you to write tests in JavaScript, making it easy to define requests, loops, checks, and thresholds.

#### Installing K6

K6 offers multiple installation methods depending on your operating system:

**Windows (using Chocolatey):**

```cmd
choco install k6
```

**Windows (manual download):** Download the latest release from GitHub (k6.io/docs/getting-started/installation) and add the executable to your system PATH.

**macOS:**

```bash
brew install k6
```

**Linux (Ubuntu/Debian/WSL):**

```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

Verify the installation by running:

```bash
k6 version
```

#### Understanding How k6 Works

A k6 script has a modular structure that mirrors the lifecycle of a load test:

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

// 1. Test configuration
export const options = {
  vus: 5, // Number of concurrent virtual users
  duration: "30s", // How long the test should run
  thresholds: {
    http_req_duration: ["p(95)<500"], // 95% of requests must respond in < 500ms
  },
};

// 2. Setup function (runs once, before VUs start)
export function setup() {
  // e.g. authenticate, prepare data
}

// 3. Main function (executed by every VU, repeatedly)
export default function () {
  const res = http.get("https://blog.example.com/");

  // Check response validity
  check(res, {
    "status is 200": (r) => r.status === 200,
  });

  // Simulate user "think time"
  sleep(1);
}

// 4. Teardown function (runs once, after all VUs finish)
export function teardown(data) {
  // e.g. cleanup, release resources
}
```

##### Virtual Users (VUs)

A **VU = one simulated user**.  
If you set `vus: 5`, you’re saying: _“Pretend five people are using my app at the same time.”_

Each VU executes the test function independently. In the example above:

- 5 users are browsing the site,

- each one requests a page, waits one second, then repeats,

- for 30 seconds straight.

  That results in about **5 requests per second** (since each user makes 1 request, then waits 1 second).

##### Duration and Stages

The `duration` tells k6 how long to run the test. But in real-world tests, you usually want traffic to change over time. That’s where **stages** come in:

```javascript
export const options = {
  stages: [
    { duration: "1m", target: 10 }, // ramp up to 10 users in 1 min
    { duration: "3m", target: 10 }, // stay at 10 users
    { duration: "1m", target: 30 }, // ramp up to 30 users
    { duration: "2m", target: 0 }, // ramp down to 0
  ],
};
```

This pattern is realistic: traffic doesn’t jump from 0 → 100 instantly. It grows, peaks, and falls off.

For example, this could simulate:

- 10 readers browsing your blog at the same time during normal hours,

- a spike of 30 users when a new article goes viral,

- and finally traffic cooling down.

##### Checks and Thresholds

- **Checks** are like **assertions inside the test**. They don’t stop the test but tell you what percentage of responses failed expectations. Example:

  ```js
  check(res, { "status is 200": (r) => r.status === 200 });
  ```

  If only 80% of responses return `200 OK`, you’ll see that in the results.

- **Thresholds** are like **test pass/fail criteria**. Example:

  ```js
  thresholds: {
    http_req_duration: ["p(95)<500"], // 95% of requests must be under 500ms
  }
  ```

  If the app can’t keep up, k6 exits with an error code. This makes thresholds perfect for CI/CD pipelines.

##### Sleep (Think Time)

Without `sleep()`, users would hammer your server non-stop. Real people don’t behave like that.  
By adding `sleep(1)`, we say: _“After loading a page, the user spends one second reading before clicking the next link.”_  
This makes the load pattern more realistic and helps uncover real-world bottlenecks.

#### Running the Test

Save the script as `blog-load-test.js` and run:

```bash
k6 run blog-load-test.js
```

k6 will start sending requests and show a live progress report in your terminal.

After the test finishes, you’ll see a summary like this:

```
     checks................: 100.00% ✓ 50 ✗ 0
     http_req_duration.....: avg=200ms  p(95)=350ms  p(99)=480ms  max=600ms
     http_req_failed.......: 0.00% ✓ 0 ✗ 50
     http_reqs.............: 50 total
     vus...................: 5
     iterations............: 50
```

Here’s how to interpret the key metrics:

- **http_req_duration**: End-to-end request time.

  - `avg=200ms` → typical page load was fast.

  - `p(95)=350ms` → 95% of users got a response in under 350ms.

  - `max=600ms` → worst-case request took 0.6s.

- **http_req_failed**: Percentage of failed requests. Ideally 0%.

- **checks**: Shows whether your assertions (like `status == 200`) passed.

- **iterations**: Number of times the test function ran (i.e. how many “user actions” happened).

Real users don’t just refresh the homepage over and over — they log in, browse, click around, and maybe submit a form. k6 can model this by chaining multiple requests per virtual user.

Here’s a more advanced script:

```js
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "1m", target: 20 }, // ramp up to 20 users
    { duration: "3m", target: 20 }, // stay at 20
    { duration: "1m", target: 0 }, // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<800"], // 95% of requests under 800ms
    http_req_failed: ["rate<0.01"], // <1% of requests can fail
  },
};

export default function () {
  // Step 1: Visit homepage
  let res = http.get("https://blog.example.com/");
  check(res, { "homepage status is 200": (r) => r.status === 200 });

  sleep(1); // user reads homepage

  // Step 2: Log in (POST request with form data)
  res = http.post("https://blog.example.com/login", {
    username: "testuser",
    password: "password123",
  });
  check(res, { "login successful": (r) => r.status === 200 });

  sleep(1); // user thinks before next action

  // Step 3: Load dashboard (authenticated)
  res = http.get("https://blog.example.com/dashboard");
  check(res, { "dashboard loaded": (r) => r.status === 200 });

  sleep(2); // user spends time browsing
}
```

What This Script Demonstrates:

- **Multiple requests per user session**  
  Each VU now behaves like a real user: homepage → login → dashboard.  
  k6 automatically handles cookies, so the session persists across requests.

- **POST requests**  
  Useful for simulating logins, form submissions, or API calls.

- **Checks on each step**  
  Ensures not only the homepage loads, but also that login and dashboard pages respond correctly.

- **More realistic think time**  
  Different `sleep()` values model how users pause between actions.

####

### c. Loader.io

[Loader.io](https://loader.io/) is a cloud-based load testing service that makes it easy to test your web applications without installing any software locally. Unlike `ab` or `k6`, which you run from your own machine, Loader.io generates traffic from their servers — giving you a better sense of how your app performs under distributed, external load.

#### Verifying Your Host

After signing up, Loader.io requires you to **prove ownership of the domain** you want to test. This step prevents abuse (e.g. testing someone else’s website). Verification is simple:

1. Loader.io gives you a unique verification token.

2. You either upload it as a plain text file to your server root, or add a DNS record.

3. Once verified, you can start running tests.

This process ensures that only domain owners can stress test a site.

#### Creating a Test

When you create a test, Loader.io gives you two main traffic generation models:

![loader.io test creation](<loader.io test creation.webp>)

##### 1. Clients Per Test

In this mode, Loader.io will simulate a fixed number of clients over the test duration. For example:

- **10,000 clients over 1 minute** → Loader.io distributes these requests evenly during the test.

- This is useful for measuring **how many total requests your app can handle** in a given timeframe.

Think of it as: _“I want to see what happens if my site gets X visits in Y seconds.”_

##### 2. Clients Per Second

In this mode, Loader.io generates a steady stream of clients every second. For example:

- **100 clients per second for 2 minutes** → your server receives a consistent 100 requests every second, totaling 12,000 requests.

- This is closer to real-world traffic spikes, such as a product launch or flash sale.

Think of it as: _“I want to see if my app can handle a continuous surge of users hitting it at the same time.”_

#### Analyzing the Results

After the test, Loader.io provides a visual report:

![loader.io test result example](<loader.io test result example.webp>)

- **Response times** (average, min, max).

- **Success vs failed requests** (timeouts, errors).

- **Throughput** (requests handled per second).

Because tests run from external servers, the results better reflect how users on the internet experience your app — including DNS resolution, SSL negotiation, and CDN behavior.

## 4. Load Testing Best Practices

Load testing is most useful when it’s realistic and structured. To get meaningful results without shooting yourself in the foot, keep these best practices in mind:

- **Test in a safe environment**: If possible, run tests on a staging server to avoid disrupting real users. If you must test production, schedule it during low-traffic hours.

- **Start small, scale gradually**: Begin with a few virtual users, then increase step by step. Sudden massive spikes rarely reflect real-world traffic.

- **Monitor the whole stack**: Use server monitoring tools (like `htop` or Beszel) alongside load testing. High response times might be caused by CPU, memory, or database bottlenecks.

- **Simulate real user behavior**: Add think times (`sleep`), realistic request flows, and different endpoints. Pure hammering doesn’t reflect how people actually use your site.

- **Define success criteria**: Decide what’s “good enough” before running a test (e.g. _95% of requests under 500ms, <1% errors_). Otherwise, the numbers won’t mean much.

- **Interpret results in context**: Handling 1,000 requests per second may sound great, but if your site only needs to support 100 concurrent readers, you may already be more than covered.

With these principles in mind, tools like Apache Bench, Loader.io, and k6 become more than just benchmarks — they help you **confidently measure, understand, and improve** how your applications perform under real-world load.
