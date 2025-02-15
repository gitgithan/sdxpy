---
syllabus:
-   The HyperText Transfer Protocol (HTTP) specifies one way to interact via messages over sockets.
-   A minimal HTTP request has a method, a URL, and a protocol version.
-   A complete HTTP request may also have headers and a body.
-   An HTTP response has a status code, a status phrase, and optionally some headers and a body.
-   "HTTP is a stateless protocol: the application is responsible for remembering things between requests."
status: "awaiting revision"
depends:
-   ftp
---

Uploading and downloading files ([%x ftp %]) is useful,
but we want to do more.
What we *don't* want to do is
create a new [%i "protocol" %] for every kind interaction.
Instead,
we would like to use a single standardized protocol in a variety of ways.

The [%g http "Hypertext Transfer Protocol" %] (HTTP)
specifies one way programs can exchange data over IP.
HTTP is deliberately simple:
the [%i "client" %] sends a [%g http_request "request" %]
specifying what it wants over a [%i "socket" %],
and the [%i "server" %] sends a [%g http_response "response" %] containing some data.
A server can construct responses however it wants;
it can copy a file from disk,
generated [%i "HTML" %] dynamically,
or almost anything else.

An HTTP request is that it's just text:
any program that wants to can create one or parse one.
An absolutely minimal HTTP request has just
a [%g http_method "method" %],
a [%g url "URL" %],
and a [%g http_protocol_version "protocol version" %]
on a single line separated by spaces:

[% inc file="minimal_http_request.txt" %]

The HTTP method is almost always either `GET` (to fetch information)
or `POST` (to submit form data or upload files).
The URL specifies what the client wants:
it is often a path to a file on disk,
such as `/index.html`,
but (and this is the crucial part)
it's completely up to the server to decide what to do with it.
The HTTP version is usually "HTTP/1.0" or "HTTP/1.1";
the differences between the two don't matter to us.

Most real requests have a few extra lines called
[%g http_header "headers" %],
which are key-value pairs like the three shown below:

[% inc file="http_request_headers.txt" %]

Unlike the keys in hash tables,
[%i "key" "keys" %] may appear any number of times in HTTP headers,
so that (for example)
a request can specify that it's willing to accept several types of content.

Finally,
the [%g http_body "body" %] of the request is any extra data associated with it,
such as form data or uploaded files.
There must be a blank line between the last header and the start of the body
to signal the end of the headers,
and if there is a body,
the request must have a header called `Content-Length`
that tells the server how many bytes to read.

An HTTP response is formatted like an HTTP request.
Its first line has the protocol,
a [%g http_status_code "status code" %]
like 200 for "OK" or 404 for "Not Found",
and a status phrase (e.g., the word "OK").
There are then some headers,
a blank line,
and the body of the response:

[% inc file="http_response.txt" %]

Constructing HTTP requests is tedious,
so most people use a library to do most of the work.
The most popular one in Python is the [requests][requests] module,
and works like this:

[% inc pat="requests_example.*" fill="py out" %]

`request.get` sends an HTTP GET request to a server
and returns an object containing the response ([%f http-lifecycle %]).
That object's `status_code` member is the response's status code;
its `content_length` member  is the number of bytes in the response data,
and `text` is the actual data—in this case, an HTML page
that we can analyze or render.

[% figure
   slug="http-lifecycle"
   img="http_lifecycle.svg"
   alt="HTTP request/response lifecycle"
   caption="Lifecycle of an HTTP request and response."
%]

Keep in mind that `requests` isn't doing anything magical.
Instead,
it is formatting a piece of text,
opening a socket connection ([%x ftp %]),
sending that text through the connection,
and then reading a response.
We will implement some of this ourselves in the exercises.

## Hello, Web {: #http-static}

We're now ready to write a simple HTTP server that will:

1.  wait for someone to connect and send an HTTP request;
2.  parse that request;
3.  figure out what to send back; and
4.  reply with an HTML page.

Steps 1, 2, and 4 are the same from one application to another,
so the [%i "Python standard library" %] has a module called `http.server`
that contains tools to do that for us.
Here's the entire server:

[% inc file="basic_server.py" %]

Let's start at the bottom and work our way up.

1.  `server_address` specifies the [%i "hostname" %] and [%i "port" %] of the server.
2.  The `HTTPServer` class takes care of parsing requests and sending back responses.
    When we construct it,
    we give it the server address and the name of the class we've written
    that handles requests the way we want—in this case, `RequestHandler`.
3.  Finally, we call the server's `serve_forever` method,
    which runs until it crashes or we stop it with Ctrl-C.

So what does `RequestHandler` do?

1.  When the server receives a `GET` request,
    it looks in the request handler for a method called `do_GET`.
    (If it gets a `POST`, it looks for `do_POST` and so on.)
2.  `do_GET` converts the text of the page we want to send back
    from characters to bytes—we'll talk about this below.
3.  It then sends a response code (200),
    a couple of headers to say what the content type is
    and how many bytes the receiver should expect,
    and a blank line (produced by the `end_headers` method).
4.  Finally, `do_GET` sends the content of the response
    by calling `self.wfile.write`.
    `self.wfile` is something that looks and acts like a write-only file,
    but is actually sending bytes to the socket connection.

If we run this program from the command line,
it doesn't display anything:

[% inc file="basic_server.sh" %]

If we then go to `http://localhost:8080` with our browser,
though,
we get this in our browser:

```
Hello, web!
```

and this in our shell:

[% inc file="basic_server.out" %]

The first line is straightforward:
since we didn't ask for a particular file,
our browser has asked for '/' (the root directory of whatever the server is serving).
The second line appears because
our browser automatically sends a second request
for an image file called `/favicon.ico`,
which it will display as an icon in the address bar if it exists.

## Serving Files {: #http-files}

Serving the same page for every request isn't particularly useful,
so let's rewrite our simple server to return files.
The basic logic looks like this:

[% inc file="file_server.py" keep="do_get" %]

We first turn the path in the URL into a local file path.
(We assume that all paths are [%g path_resolution "resolved" %]
relative to the directory that the server is running in.)
If that path corresponds to a file, we send it back to the client.
If nothing is there,
or if what's there is not a file,
we send an error message.
{: .continue}

It might seem simpler to rewrite `do_GET` to use `if`/`else` instead of `try`/`except`,
but the latter has an advantage:
we can handle errors that occur inside methods we're calling (like `handle_file`)
in the same place and in the same way as we handle errors that occur here.
This approach is sometimes called [%g throw_low_catch_high "throw low, catch high" %],
which means that errors should be flagged in many places
but handled in a few places high up in the code.
The method that handles files is an example of this:

[% inc file="file_server.py" keep="handle_file" %]

If there's an error at any point in the processing cycle
we send a page with an error message
*and* an error status code:

[% inc file="file_server.py" keep="handle_error" %]

The error page is just HTML with some placeholders for the path and message:

[% inc file="file_server.py" keep="error_page" %]

The code that actually sends the response is similar to what we've seen before:
{: .continue}

[% inc file="file_server.py" keep="send_content" %]

This server works, but only for a very forgiving definition of "works".
We are careful not to show clients the actual paths to files on the server
in our error messages,
but if someone asked for `http://localhost:8080/../../passwords.txt`,
this server will happily look two levels up from the directory where it's running
and try to return that file.
The server machine's passwords probably aren't stored there,
but with enough `..`'s and some patience,
an attacker could poke around large parts of our filesystem.
We will tackle this security hole in the exercises.

Another problem is that `send_content` always tells clients that
it is returning an HTML file with the `Content-Type` header.
It should instead look at the extension on the file's name
and set the content type appropriately,
e.g.,
return `image/png` for a PNG-formatted image.

One thing the server is doing right is [%i "character encoding" %].
The `send_content` method expects `content` to be a `bytes` object,
not a string,
because the HTTP protocol requires the content length to be the number of bytes.
The server reads files in [%i "binary mode" %]
by using `"rb"` instead of just `"r"` when it opens files in `handle_file`,
converts the internally-generated error page from characters to bytes
using the [%i "UTF-8" %] encoding,
and specifies `charset=utf-8` as part of the content type.

## Testing  {: #http-testing}

As with the server in [%x ftp %],
we can work backward from a test we want to be able to write
to create a testable server.
We would like to create a file,
simulate an HTTP GET request,
and check that the status, headers, and content are correct.
[%f http-inheritance %] shows the final [%i "inheritance" %] hierarchy:

-   `BaseHTTPRequestHandler` comes from the [%i "Python standard library" %].

-   `MockRequestHandler` defines replacements for its method.

-   `ApplicationRequestHandler` contains our server's logic.

-   `RequestHandler` combines our application code
    with Python's request handler.

-   `MockHandler` combines it with our mock request handler.

It's a lot of work to test a GET request for one file,
but we can re-use `MockRequestHandler` to test
the application-specific code for other servers.
Most libraries don't provide helper classes like this to support testing,
but programmers appreciate those that do.

[% figure
   slug="http-inheritance"
   img="inheritance.svg"
   alt="Testing class hierarchy"
   caption="Class hierarchy for a testable server."
%]

`MockRequestHandler` is just a few lines of code,
though it would be longer if our application relied on
more methods from the library class we're replacing:

[% inc file="mock_handler.py" %]

The application-specific class contains the code we've already seen:

[% inc file="testable_server.py" keep="handler" omit="skip" %]

`MockHandler` handles the simulated request
and also stores the values that the client would receive:

[% inc file="test_testable_server.py" keep="example" %]

The main body of our runnable server
combines the two classes to create what it needs,
instantiates the result,
and runs it:

[% inc file="testable_server.py" keep="main" %]

Our tests,
on the other hand,
create a server with mocked methods:
{: .continue}

[% inc file="test_testable_server.py" keep="combined" %]

## Summary {: #http-summary}

[% figure
   slug="http-concept-map"
   img="concept_map.svg"
   alt="HTTP concept map"
   caption="HTTP concept map."
   cls="here"
%]

## Exercises {: #http-exercises}

### Parsing HTTP Requests {: .exercise}

Write a function that takes a list of lines of text as input
and parses them as if they were an HTTP request.
The result should be a dictionary with the request's method,
URL,
protocol version,
and headers.

### Query Parameters {: .exercise}

A URL can contain [%g query_parameter "query parameters" %].
Read the documentation for the [urlparse][py_urlparse] module
and then modify the file server example so that
a URL containing a query parameter `bytes=N`
(for a positive integer N)
returns the first N bytes of the requested file.

### Better Path Resolution {: .exercise}

Modify the file server so that:

1.  it must be given the absolute path to a directory
    as a command-line argument
    when started; and

2.  it only serves files in or below that directory
    (so that paths containing `..` and other tricks
    can't be used to retrieve arbitrary files).

### Better Content Types {: .exercise}

Read the documentation for the [mimetypes][py_mimetypes] module
and then modify the file server to return the correct content type
for files that aren't HTML (such as images).

### Uploading Files {: .exercise}

Modify the file server to handle POST requests.

1.  The URL must specify the name of the file being uploaded.

2.  The body of the request must be the bytes of the file.

3.  All uploaded files are saved in a single directory,
    i.e.,
    upload paths cannot contain directory components.

### Checking Content Length {: .exercise}

Modify the file server so that:

1.  if the client sends more content than indicated in the `Content-Length` header,
    the extra bytes are read but ignored; and

2.  if the client sends less content,
    the server doesn't wait indefinitely for the missing bytes.

What status code should the server return to the client in each case?

### Directory Listing {: .exercise}

1.  Modify the file server so that
    if the path portion of the URL identifies a directory,
    the server returns a plain text list of the directory's contents.

2.  Write tests for this using the [pyfakefs][pyfakefs] module.

### Dynamic Results {: .exercise}

Modify the file server so that if the client requests the "file" `/time`,
the server returns an HTML page that reports the current time on the server's machine.

### Templated Results {: .exercise}

Modify the file server to:

1.  turn the query parameter's in the URL into a dictionary;

2.  use that dictionary to fill in a template page ([%x template %]); and

3.  return the resulting HTML page to the client.
