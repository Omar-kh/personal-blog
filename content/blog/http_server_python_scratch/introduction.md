---
title: "The road begins: HTTP, Sockets and Hello World"
date: 2022-11-25
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

It's a cool autumn evening. I'm sitting at my desk, a steaming cup of tea by my side, and a soft jazz playlist filling the air with its soothing notes. I've just closed the tab on a captivating article about some obscure facet of technology, and I find myself contemplating the unsung heroes of the digital world: HTTP servers. They're like the stagehands in a grand theater production, quietly making sure everything runs without a hitch, yet rarely receiving the applause.

At that moment, fingers lightly drumming on the keyboard, an idea strikes me like a bolt of lightning. Why not build an HTTP server from scratch? The challenge, the thrill of discovery, and the uncharted territory beckon me.

Richard Feynman's wisdom echoes in my thoughts:

> "What I cannot create, I do not understand."

As a developer who has been deploying these servers as the backbone of my applications, I realize it's high time to pull back the curtain. To dissect, explore, and reconstruct what has always been a tool but has the potential to be a teacher. So here I go, venturing into the labyrinthine world of HTTP, notebook and code editor at the ready.

## A brief story about HTTP

### From the 1980s to the modern days

It's the 1980s, and the academic world is abuzz with excitement. Computers, once the size of entire rooms and accessible only to a select few, are finding their way into research institutions across the globe. But there's a challenge. These institutions want to share research, data, and insights, yet there's no universal way to do so seamlessly. Enter Tim Berners-Lee, a British computer scientist who's about to change the landscape forever.

Imagine being Tim for a moment. You're presented with a plethora of disparate networks and systems, each with its own set of protocols. Your goal is simple yet profoundly ambitious: create a unified system to facilitate easy and efficient data exchange. The World Wide Web is born, and at its heart lies the **Hypertext Transfer Protocol** or **HTTP**.

At its core, HTTP was designed as a stateless application-layer protocol to allow the exchange of hypertext documents, those with links that could lead you from one document to another. The initial version, HTTP/0.9, was simple. You, as the client, made a single-line request to a server, and it replied with a hypertext document. For instance:

```plaintext
Client: GET /index.html
Server: <HTML>...content...</HTML>
```

That was it! No headers, no metadata, just the request and the response. But the internet was evolving rapidly. The simplicity of HTTP/0.9 made it incredibly adaptable but also limited in capabilities.

Soon, HTTP/1.0 took shape, introducing a new feature: headers. These headers allowed both clients and servers to exchange additional information. Suddenly, it wasn't just about fetching a document; it was about understanding the nature, type, language, and encoding of that document:

```plaintext
Client:
GET /index.html HTTP/1.0
User-Agent: Mozilla/4.0

Server:
HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 137
<HTML>...content...</HTML>
```

As you can observe, the request and response now had structured headers. This was a leap from its predecessor, setting the stage for a more versatile and robust communication.

But, with increased web traffic and more complex websites, HTTP/1.0's limitations became evident. For every single resource—a CSS file, an image, a script—a new connection had to be made, adding overhead and latency. Enter HTTP/1.1. Persistent connections were introduced, allowing multiple requests and responses in a single connection. Caching mechanisms were enhanced, ensuring not every request led to a server fetch, reducing bandwidth usage.

The world beneath HTTP, the one that ensured data packets traveled reliably, was the **Transmission Control Protocol (TCP)**. When you, as a hypothetical browser, sent an HTTP request, it got translated into multiple TCP packets, ensuring data integrity and ordered delivery. Think of it like this: if HTTP was the message in a letter, TCP was the postal system ensuring the letter got to its destination correctly and in order.

However, with the rise of multimedia content and the need for even faster data exchange, HTTP/2 emerged. It introduced multiplexing, allowing multiple messages to be sent at once without waiting for the first to finish. This was like sending several letters at once, even if you hadn't received a reply to your first one yet.

From the 1980s to the present, the journey of HTTP has been about adapting to the needs of the time, always ensuring the web remained a space of seamless information exchange.

### Diving deeper in the Requets / Responses cycle

In the early days of HTTP, it wasn't merely about sending a request and awaiting a response. The protocol has nuanced layers, often overshadowed by its primary functionality. Let's quickly delve into these intricate parts.

**Headers: The Unsung Communicators**

HTTP headers are like a coded handshake between client and server, exchanging essential metadata.

```plaintext
GET /my-page HTTP/1.1
Host: www.example.com
User-Agent: MyBrowser/1.0
Accept: text/html
```

Headers such as User-Agent describe the client's software, while Accept indicates acceptable media types. Others, like ETag or Cache-Control, manage efficient content updates or caching directions, respectively.

**HTTP Methods: Beyond the Basics**

While most are familiar with `GET` and `POST`, HTTP boasts a diverse array of methods for various operations. As an example:

- `PUT` updates or even creates resources.
- `DELETE` removes specified resources.
- `OPTIONS` inquires about permissible methods for a resource.

These methods, among others, enable developers to craft applications with richer functionality.

**Status Codes: Web's Expressive Language**

Web servers often communicate using status codes. For instance, `200 OK` signals all's well, and `404 Not Found` denotes missing content.

```plaintext
HTTP/1.1 200 OK
Date: Sat, 25 Sep 2023 23:59:59 GMT
```

But there are quirkier ones: `429 Too Many Requests` warns about over-eagerness, while `418 I'm a teapot` humorously claims the server is a teapot.

In sum, HTTP isn't just about primary requests and responses. It's a choreographed dance of metadata, methods, status codes, and efficient connections, operating behind the scenes of our web interactions.

### More Resources for the curious minds

Diving deeper into HTTP's intricacies? Here are a few invaluable resources to propel your journey:

- **RFCs (Request For Comments)**: The official documentation for internet standards. You can start with [RFC 2616](https://tools.ietf.org/html/rfc2616) for HTTP/1.1 specifics.
- **Mozilla Developer Network (MDN)**: An extensive [guide on HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP) with detailed explanations and examples.
- **W3C Specifications**: The World Wide Web Consortium's [official documentation](https://www.w3.org/Protocols/) on web standards, including HTTP.

## Setting the Stage for Building Our Own HTTP Server

Rolling out our own rudimentary HTTP server using Python’s socket library can be an illuminating journey. This foundational example will attempt to demystify the dance between HTTP and its underlying protocol, TCP.

**Simple HTTP Server with Python**

At its simplest, writing a barebones HTTP server using the `socket` library in Python would resemble something like this:

```python
import socket

HOST = '127.0.0.1'
PORT = 8080
QUEUE_SIZE = 3

# Create a new socket using the given address family and socket type
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.bind((HOST, PORT))
server_socket.listen(QUEUE_SIZE)
print(f"Server listening on {HOST}:{PORT}")

while True:
    client_socket, address = server_socket.accept()
    request = client_socket.recv(1024).decode('utf-8')
    print(f"Received request:\n{request}")
    response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello, World!"
    client_socket.sendall(response.encode())
    client_socket.close()
```

Let's go through it, step-by-step.

In the first statements, we've imported the socket module and defined the host and port we want our server to run on. The `socket.socket()` method initializes a new socket (or endpoint) for network communication. This is where the magic starts:

- `socket.AF_INET`: This is the address family for the socket. `AF_INET` corresponds to IP version 4 (IPv4). There's also `AF_INET6` for IP version 6, but for simplicity, we're sticking with IPv4. The choice of address family determines what kind of addresses can be used with this socket and what kind of network it can speak over.

- `socket.SOCK_STREAM`: This is the type of socket we're creating. `SOCK_STREAM` means it's a TCP socket. TCP (Transmission Control Protocol) is one of the main transport protocols in the IP suite and is connection-based, guaranteeing the delivery of packets in the order they were sent. This makes it suitable for transmitting things like HTTP, where the order and integrity of packets is important. If we were working with a protocol like UDP (User Datagram Protocol), which is connectionless and doesn't guarantee the order of delivery, we'd use `SOCK_DGRAM` instead.

Choosing `AF_INET` and `SOCK_STREAM` means we're creating a TCP socket that communicates using IPv4 addresses.

```python
import socket

HOST = '127.0.0.1'
PORT = 8080
QUEUE_SIZE = 3

# Create a new socket using the given address family and socket type
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

Next, we associate the socket with a specific address and port on the machine using the `bind()` method. In our case, we're binding it to `127.0.0.1`, the loopback address, meaning the socket will listen for connections originating from the local machine.

```python
server_socket.bind((HOST, PORT))
```

With `listen(QUEUE_SIZE)`, our socket is now marked as "listening". The number 5 indicates the maximum number of queued connections our server will allow.

```python
server_socket.listen(QUEUE_SIZE)
```

The while loop ensures our server continuously accepts new connections. `accept()` waits for an incoming connection and returns a new socket representing the client, and the address of the client.

```python
print(f"Server listening on {HOST}:{PORT}")

while True:
    client_socket, address = server_socket.accept()
```

When working with sockets, it's critical to remember that data is transmitted over networks as bytes. This means that when we receive data from a client using the `recv()` method, the data comes in as bytes. In the code snippet below, we are doing two things:

- Receiving data (in bytes) from the client (`recv(1024)`).
- Decoding the byte data to a string using the UTF-8 character encoding (`decode('utf-8')`).

```python
    request = client_socket.recv(1024).decode('utf-8')
    print(f"Received request:\n{request}")
```

Similarly, when we're ready to send data back to the client, we formulate a basic HTTP response, convert (or encode) it into bytes using the UTF-8 character encoding before sending, and then send it back to the client using `sendall()`.
After that, we close the client socket, effectively ending our communication with that client.

```python
    response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello, World!"
    client_socket.sendall(response.encode())
    client_socket.close()
```

Note that the code example provided will send a response regardless of the request's validity. In a real-world scenario, this could lead to a plethora of issues, from security vulnerabilities to unexpected behaviors.

To ensure that our server is responding appropriately, we should validate the HTTP request. At a basic level, this could be as simple as checking that the request starts with `GET`:

```python
if request.startswith("GET"):
    response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello, World!"
else:
    response = "HTTP/1.1 400 Bad Request\r\nContent-Type: text/plain\r\n\r\nInvalid request!"
```

Resulting in the following code:

```python {hl_lines=["15-20"]}
import socket

HOST = '127.0.0.1'
PORT = 8080
QUEUE_SIZE = 3

# Create a new socket using the given address family and socket type

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.bind((HOST, PORT))
server_socket.listen(QUEUE_SIZE)
print(f"Server listening on {HOST}:{PORT}")

while True:
  client_socket, address = server_socket.accept()
  request = client_socket.recv(1024).decode('utf-8')

  if request.startswith("GET"):
    response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello, World!"
  else:
    response = "HTTP/1.1 400 Bad Request\r\nContent-Type: text/plain\r\n\r\nInvalid request!"

  print(f"Received request:\n{request}")
  response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello, World!"
  client_socket.sendall(response.encode())
  client_socket.close()
```

With this rudimentary validation, we're only accepting `GET` requests and rejecting any other types of requests with a `400 Bad Request` response. In reality, HTTP validation would be much more complex and might involve checking headers, the request body, or other aspects of the HTTP request.

**Testing with a CLI Client**

Now, with our server running, we can interact with it using `telnet`, a popular command-line client:

```bash
$ telnet 127.0.0.1 8080
```

This telnet session establishes a TCP connection, letting you manually send an HTTP request. Our server then processes the request and returns the response.
After connecting and typing:

```http
GET / HTTP/1.1
```

We get to see the `Hello, World!` response from our server!

**Behind the Curtain: The socket Library**

This seemingly simple socket library is doing a ton of work behind the scenes. By providing an interface to the lower-level C socket API, it's allowing us to interact with the network stack directly. It manages buffer space, handles network errors, and much more.

## A Glimpse Ahead

While we've touched upon the foundational aspects of sockets and HTTP in this article, the journey is far from over. We'll be diving deeper into the essence of networking, from the lifecycle of a socket to the UNIX philosophy that seamlessly interweaves the concept of files and sockets. Additionally, I plan to unearth the subtleties behind how the system actually "listens" to these sockets and the magic that occurs behind the scenes.
