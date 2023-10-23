---
title: "Rolling a minimal Gunicorn: WSGI and connection management"
date: 2023-01-05
categories:
  - DeepDive
tags:
  - web
  - http
  - network
  - python
series:
  - Building a Python HTTP server
---

The [simple HTTP server we built previously]({{< relref "introduction.md#setting-the-stage-for-building-our-own-http-server" >}}) was a great starting point, but it's limited when it comes to handling concurrent connections. Enter [Gunicorn](https://gunicorn.org)—a renowned WSGI HTTP server designed for serving Python applications. In this article, I'll craft a stripped-down version of Gunicorn, delving into the intricacies of the [Web Server Gateway Interface (WSGI)](https://wsgi.readthedocs.io/en/latest/what.html) and the art of asynchronous connection management, both essential for modern web applications.

## WSGI: The Building Block

### A brief overview

WSGI is a specification, a set of rules and standards that defines how web servers interact with web applications or frameworks in the Python realm. Before WSGI came into the picture, there was a significant gap. Each web framework had its own way of communicating with servers, leading to a fragmented landscape. Deploying applications could become a chore, as engineers had to ensure that their chosen framework and server were compatible.

The introduction of WSGI provided a common ground—a universal interface, if you will. To offer a metaphor, think of WSGI as a universal plug adapter during your international travels. Each country might have its socket style, representing individual web servers or frameworks. Your device, perhaps representing your web application, needs a way to plug into these different sockets seamlessly. The universal adapter (WSGI) ensures that your device can work efficiently regardless of where you are. It unifies, simplifies, and streamlines.

### Building the WSGI handler

Now for the fun part. Consider the following code snippet:

```python
import socket
from io import BytesIO
import sys
import importlib
response_headers = []


def parse_request(data):
    headers = data.split("\r\n")
    request_line = headers[0].split()
    method = request_line[0]
    path = request_line[1]
    return {
        'REQUEST_METHOD': method,
        'PATH_INFO': path,
        'QUERY_STRING': '',
        'wsgi.input': BytesIO(data.encode()),
        'wsgi.version': (1, 0),
        'wsgi.url_scheme': 'http',
    }


def start_response(status, headers):
    response_headers[:] = [status, headers]


def wsgi_handler(client_socket, app):
    request_data = client_socket.recv(1024).decode("utf-8")
    environ = parse_request(request_data)
    response_body = app(environ, start_response)
    send_response(client_socket, response_body)
    client_socket.close()


def send_response(client_socket, response_body):
    response = "HTTP/1.1 " + response_headers[0] + "\r\n"
    for header in response_headers[1]:
        response += f"{header[0]}: {header[1]}\r\n"
    response += "\r\n"
    for data in response_body:
        response += data.decode("utf-8")
    client_socket.sendall(response.encode())


def serve(app, host="127.0.0.1", port=8000):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind((host, port))
    server_socket.listen(1)
    print(f"Serving on {host}:{port}")

    while True:
        client_socket, addr = server_socket.accept()
        wsgi_handler(client_socket, app)


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python minimal_gunicorn.py <module_name>:<app_instance>")
        sys.exit(1)

    module_name, app_instance_name = sys.argv[1].split(":")
    module = importlib.import_module(module_name)

    if hasattr(module, app_instance_name):
        app_instance = getattr(module, app_instance_name)
        serve(app_instance)
    else:
        print(f"No '{app_instance_name}' found in {module_name}.")
        sys.exit(1)
```

**Parsing the Request**

Before the server can converse with the application using the WSGI protocol, it needs to understand the incoming request's language.
In `parse_request`, we're translating a raw HTTP request into a WSGI-compatible `environ` dictionary. This dictionary acts as an interface, allowing the application (in this case, Flask) to understand the request.

**Crafting the Response**

The `start_response` function provides a way for the application to communicate back to the server. It's designed to be passed as a callable to the application, letting the app dictate the status and headers of the HTTP response.

**Handling the WSGI Request**

The server's primary responsibility is to facilitate a conversation between the client and the Python application. The `wsgi_handler` is where this dialogue comes together.

```python
def wsgi_handler(client_socket, app):
    request_data = client_socket.recv(1024).decode("utf-8")
    environ = parse_request(request_data)
    response_body = app(environ, start_response)
    send_response(client_socket, response_body)
    client_socket.close()
```

Here's the step-by-step flow:

1. Receive data from the client.
2. Parse this data into a WSGI-compliant format.
3. Pass the data to the app, along with the `start_response` callable.
4. Once the app processes the request and determines an appropriate response, we'll compile and send that response back to the client.

**Serving It All Up**

The WSGI-compliant server is still a server at its core. It listens for incoming connections, and for each connection, it facilitates this server-app dialogue we've been crafting.

```python
def serve(app, host="127.0.0.1", port=8000):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind((host, port))
    server_socket.listen(1)
    print(f"Serving on {host}:{port}")

    while True:
        client_socket, addr = server_socket.accept()
        wsgi_handler(client_socket, app)
```

Having a WSGI-compliant server in place allows to run Python web applications, regardless of their specifics.
As an example, the following basic Flask application:

```python
from flask import Flask

app = Flask(__name__)


@app.route('/')
def hello():
    return "Hello, World!"
```

can be run with the minimal gunicorn script above like so:

```bash
python minimal_gunicorn.py flask_app:app
```

This implementation works for basic needs. However, as the web applications scale and the demand for concurrent requests increases, it will soon face bottlenecks.
As a result, it is necessary to implement a concurrency mechanism to handle the numerous incoming requests.

## Introducing Concurrent Handling

Handling multiple connections, particularly in a web server context, can quickly become a challenge. Let's unpack why.

### The Limitation of Sequential Handling

Imagine you're at a coffee shop, and there's just one barista. Every customer must wait for the previous one to receive their coffee before being served. Now, if a customer orders a complex latte art or some other time-consuming drink, the line grows and everyone waits. This scenario is similar to how a simple, non-asynchronous server handles connections — sequentially.

In the web server realm, such a delay is even more pronounced. Some requests might involve complex database queries, third-party service calls, or other time-consuming tasks. If the server handles each request one after the other, it means users experience delays, and the system becomes inefficient.

### Concurrent Handling to the Rescue

Enter concurrent handling.

Imagine now that the coffee shop has multiple baristas or, better yet, an automated system that can prepare several orders simultaneously. Customers get their coffee faster, and the shop can handle a rush hour effectively.

Similarly, by introducing either multi-threading or asynchronous I/O to the server, it becomes possible to process multiple connections simultaneously. While one request is waiting for a database query, another can be served, and another can be processed, all in the same time slice. It's efficient, scalable, and essential for any modern web application or service.

### Code Implementation

Let's introduce basic multi-threading to the server:

```python
import socket
from io import BytesIO
import sys
import importlib
from threading import Thread

# ... [Rest of the code stays the same]

def serve(app, host="127.0.0.1", port=8000):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind((host, port))
    server_socket.listen(1)
    print(f"Serving on {host}:{port}")

    while True:
        client_socket, addr = server_socket.accept()
        Thread(target=wsgi_handler, args=(client_socket, app)).start()  # Use threading here

# ... [Rest of the code stays the same]
```

By using Python's threading module, a new thread is spawned for each incoming connection. This ensures that even if one request is taking a long time, other requests can be processed in parallel. However, it's essential to understand that while threading does introduce concurrency, it's not always the most efficient for I/O-bound tasks due to Python's Global Interpreter Lock (GIL). In such cases, asynchronous I/O libraries like `asyncio` could offer better performance.

## The Gunicorn-esque Touch

Gunicorn, or the "Green Unicorn", is widely admired for its efficiency, stability, and the array of features that make it an attractive choice for production deployments. At its heart, two features, in particular, stand out — the use of **worker processes** and the ability for **graceful shutdowns**. Let's break these down.

### Worker Processes

One of the primary reasons Gunicorn can handle a large number of simultaneous connections is its ability to fork multiple worker processes. Each worker process is a separate instance of your application, allowing it to process requests independently of others. Think of worker processes as multiple baristas in our coffee shop example from earlier — the more you have (up to a point), the more customers you can serve at once.

In essence, worker processes allow Gunicorn to parallelize request handling, overcoming some of the limitations of the Global Interpreter Lock (GIL) in CPython.

**Code Implementation:**

To introduce a simplistic version of worker processes, we'll leverage Python's `multiprocessing` capabilities:

```python
import socket
from io import BytesIO
import sys
import importlib

# ... [rest of the previous code stays the same]

def run_server(app, host="127.0.0.1", port=8000, num_workers=4):

    # Step 1: Create and bind the server socket in the parent process
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind((host, port))
    server_socket.listen(10)

    processes = []

    for _ in range(num_workers):
        pid = os.fork()
        if pid == 0:  # Child process
            serve(app, server_socket)  # Modified serve function to accept server_socket
            os._exit(0)
        else:
            processes.append(pid)

    # Parent process waits for all child processes to complete
    for pid in processes:
        os.waitpid(pid, 0)


def serve(app, server_socket):
    print(f"Worker {os.getpid()} is ready to serve!")
    while True:
        client_socket, addr = server_socket.accept()
        wsgi_handler(client_socket, app)


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python minimal_gunicorn.py <module_name>:<app_instance>")
        sys.exit(1)

    module_name, app_instance_name = sys.argv[1].split(":")
    module = importlib.import_module(module_name)

    if hasattr(module, app_instance_name):
        app_instance = getattr(module, app_instance_name)
        run_server(app_instance)
    else:
        print(f"No '{app_instance_name}' found in {module_name}.")
        sys.exit(1)
```

Notice the `pid = os.fork()`. This is the part where all the magic happens!
Forking is a feature of the underlying operating system (unix) that allows a process to make a duplicate of itself!
As this is a pretty challenging concept, I personally had to stop and do some research to understand it. One of the best explanations I have seen is [this Stack Overflow post about it](https://stackoverflow.com/questions/33560802/pythonhow-os-fork-works).

### Graceful Shutdown

Ensuring the server can shut down without abruptly cutting off active connections is crucial. Gunicorn handles this gracefully, waiting for active connections to close while not accepting any new ones.

**Code Implementation:**

Implementing a basic version of this requires handling SIGINT or SIGTERM signals to notify the server to stop accepting new connections and wrap up ongoing ones.

```python
import signal

# ... [rest of the previous code stays the same]

def signal_handler(signum, frame):
    print("Received shutdown signal. Initiating graceful shutdown...")
    # Graceful shutdown code will be placed here
    sys.exit(0)

def run_server(app, host="127.0.0.1", port=8000, num_workers=4):
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    # ... [rest of the previous code stays the same]

```

### Considerations and Trade-offs

In crafting this minimal Gunicorn-esque server, several considerations were made:

1. **Simplicity over Feature Completeness:** We aimed for a server that captures the essence of Gunicorn's approach, without delving into its plethora of optimizations and features.
2. **Performance:** While multiprocessing provides a degree of parallelism, it may not be as efficient as Gunicorn's more refined handling using various worker types.
3. **Error Handling:** Our version omits detailed error handling, logging, and other robustness features found in Gunicorn.
4. **Resource Consumption:** Forking multiple processes can be resource-intensive, which might not be ideal for all environments.

Remember, while our simplistic server captures the spirit of Gunicorn's architecture, it's meant for educational purposes. Before deploying such a system in production, it would need rigorous testing, optimization, and likely more features to ensure stability and performance.

## Testing and Performance Benchmarks

The code we've written so far represents a considerable learning journey. From a basic HTTP server to an advanced Gunicorn-esque server, the transformation has been inspiring. But, as any seasoned engineer would rightly ask: "How well does it perform?" Let's explore this very question.

To benchmark our servers, let's use a tool like [wrk](https://github.com/wg/wrk). It's a modern HTTP benchmarking tool capable of generating significant load when run on a single multi-core CPU:

```bash
sudo apt install wrk
```

Let's run our servers:

### No Concurrency

```bash
python minimal_gunicorn.py flask_app:app
```

then

```bash
wrk -t4 -c100 -d10s http://localhost:8000
```

It outputs:

```bash
Running 10s test @ http://localhost:8000
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.75ms  115.40ms   1.89s    98.41%
    Req/Sec     5.20k     3.68k   12.98k    63.81%
  113554 requests in 10.03s, 9.96MB read
  Socket errors: connect 0, read 113554, write 0, timeout 5
Requests/sec:  11317.57
Transfer/sec:      0.99MB
```

### Multi Workers Model

The results

```bash
Running 10s test @ http://localhost:8000
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    19.69ms  132.98ms   1.68s    97.27%
    Req/Sec     8.81k     4.36k   22.49k    73.04%
  335572 requests in 10.03s, 29.44MB read
  Socket errors: connect 0, read 335567, write 0, timeout 13
Requests/sec:  33470.15
Transfer/sec:      2.94MB
```

### Comparing The Results

The numbers speak for themselves. The multi-worker setup clearly has the edge, handling a lot more traffic in the same amount of time. This little experiment shows how a bit of tinkering and understanding can lead to significant improvements. It's always good to see the fruits of our labor in such concrete terms. It makes all the deep dives and head-scratching moments worth it!

## Conclusion

I hope this journey has been as enlightening for you as it has been for me. We began with a basic HTTP server, a straightforward mechanism echoing our web requests. Step by step, feature by feature, we evolved it into a more powerful tool, integrating WSGI compliance, asynchronous handling, and multi-process functionalities inspired by Gunicorn. It's akin to upgrading from a bicycle to a motorbike, learning each part's role as we add it on.

Truly knowing how something works, under the hood, equips us better for the unforeseen challenges in our engineering journey. It's not always about building from scratch, for sure, but the insight we gain from such exercises is invaluable.

## Additional Resources

To further feed your insatiable curiosity and to dig even deeper into the topics we've discussed, here are some resources I'd recommend:

- **WSGI Deep Dive**: For those who want to understand every nook and cranny of the WSGI specification, [WSGI Documentation](https://wsgi.readthedocs.io/en/latest/) is an excellent place to start.
- **Gunicorn's Source Code**: For the brave souls out there, [Gunicorn's GitHub repository](https://github.com/benoitc/gunicorn) offers an opportunity to see the real-world implementation of a production-ready server.

- **Concurrency in Python**: Dive into [Python's official documentation on concurrent execution](https://docs.python.org/3/library/concurrency.html). It provides a comprehensive look at threads, processes, and asynchronous programming models.

- **HTTP Fundamentals**: [Mozilla's HTTP Guide](https://developer.mozilla.org/en-US/docs/Web/HTTP) on MDN provides a great walkthrough of the protocol's basics, covering everything from messages to cookies to caching.

- **Wrk**: For those who want to push their servers to the limit and beyond, check out [wrk's official GitHub repository](https://github.com/wg/wrk) for installation, usage tips, and more.

I hope these resources serve you well in your continuous journey of exploration and learning. The world of web servers and HTTP is vast, but with the right guides, it's an incredible terrain to traverse. Safe travels and happy coding!
