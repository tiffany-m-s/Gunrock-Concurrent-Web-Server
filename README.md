# Purpose
This web server is a simple server from UC Davis ECS 150 for learning about
multi-threaded programming and operating systems. The original version of the
server could only handle one client at a time and simply served static
files. Also, it closed each connection after reading the request
and responding, but generally is still HTTP 1.1 compliant.

The original README file was written by Sam King from UC Davis. I then modified
it to clarify the purpose of its inclusion on Github. The original files
were uploaded and kept on Google Drive then reuploaded to Github, though the 
earlier versions of FileService.cpp went missing.

# Credit
The original server was written by Sam King from UC Davis. The 
`http_parse.c` file was written by [Ryan Dahl](https://github.com/ry) 
and is licensed under the BSD license by Ryan. This programming 
assignment is from the [OSTEP](http://ostep.org) textbook.

# Quickstart
To compile and run the server, open a terminal and execute the following commands:
```bash
$ cd project3
$ make
$ ./gunrock_web
```

To test it out, you can either open up a web browser on the same machine and give it this url `http://localhost:8080/hello_world.html` or if you want to use curl on the command line you can test it out like this:
```bash
$ # get a basic HTML file
$ curl http://localhost:8080/hello_world.html
$ # get a basic HTML file with more detailed information
$ curl -v http://localhost:8080/hello_world.html
$ # head a basic HTML file
$ curl --head http://localhost:8080/hello_world.html
$ # test out a file that does not exist (404 status code)
$ curl -v http://localhost:8080/hello_world2.html
$ # test out a POST, which isn't supported currently (405 status code)
$ curl -v -X POST http://localhost:8080/hello_world.html
```

Full website for testing, try pointing your browser to: `http://localhost:8080/bootstrap.html`

# Overview

In this assignment, you will be developing a concurrent web server. To
simplify this project, we are providing you with the code for a non-concurrent
(but working) web server. This basic web server operates with only a single
thread; it will be your job to make the web server multi-threaded so that it
can handle multiple requests at the same time.

The goals of this project are:
- To learn the basic architecture of a simple web server
- To learn how to add concurrency to a non-concurrent system
- To learn how to read and modify an existing code base effectively

Useful reading from [OSTEP](http://ostep.org) includes:
- [Intro to threads](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-intro.pdf)
- [Using locks](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf)
- [Producer-consumer relationships](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf)
- [Server concurrency architecture](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-events.pdf)


# A Basic Web Server

When you run this basic web server, you need to specify the port number that
it will listen on; ports below number 1024 are *reserved* (see the list
[here](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml))
so you should specify port numbers that are greater than 1023 to avoid this
reserved range; the max is 65535. Be wary: if running on a shared machine, you
could conflict with others and thus have your server fail to bind to the
desired port. If this happens, try a different number!

When you then connect your web browser to this server, make sure that
you specify this same port. For example, assume that you are running on
`bumble21.cs.wisc.edu` and use port number 8003; copy your favorite HTML file
to the directory that you start the web server from. Then, to view this file
from a web browser (running on the same or a different machine), use the url
`bumble21.cs.wisc.edu:8003/favorite.html`. If you run the client and web
server on the same machine, you can just use the hostname `localhost` as a
convenience, e.g., `localhost:8003/favorite.html`.

To make the project a bit easier, you are originally provided with a minimal web
server, consisting of only a few hundred lines of C++ code. As a result, the
server is limited in its functionality; it does not handle any HTTP requests
other than `GET` and `HEAD`, and understands only a few content types. This web server is also
not very robust; for example, if a web client closes its connection to the
server, it may trip an assertion in the server causing it to exit. It is not
expected that you to fix these problems.

# Instructions

In this project, you will be adding one key piece of functionality to the
basic web server: you will make the web server multi-threaded. 
You will also be modifying how the web server is invoked so
that it can handle new input parameters (e.g., the number of threads to
create).

## Part 1: Multi-threaded
 
The basic web server that we provided has a single thread of
control. Single-threaded web servers suffer from a fundamental performance
problem in that only a single HTTP request can be serviced at a time. Thus,
every other client that is accessing this web server must wait until the
current http request has finished; this is especially a problem if the current
HTTP request is a long-running CGI program or is resident only on disk (i.e.,
is not in memory). Thus, the most important extension that you will be adding
is to make the basic web server multi-threaded.

The simplest approach to building a multi-threaded server is to spawn a new
thread for every new http request. The OS will then schedule these threads
according to its own policy. The advantage of creating these threads is that
now short requests will not need to wait for a long request to complete;
further, when one thread is blocked (i.e., waiting for disk I/O to finish) the
other threads can continue to handle other requests. However, the drawback of
the one-thread-per-request approach is that the web server pays the overhead
of creating a new thread on every request.

Therefore, the generally preferred approach for a multi-threaded server is to
create a fixed-size *pool* of worker threads when the web server is first
started. With the pool-of-threads approach, each thread is blocked until there
is an http request for it to handle. Therefore, if there are more worker
threads than active requests, then some of the threads will be blocked,
waiting for new HTTP requests to arrive; if there are more requests than
worker threads, then those requests will need to be buffered until there is a
ready thread.

In your implementation, you must have a main thread that begins by creating
a pool of worker threads, the number of which is specified on the command
line. Your main thread is then responsible for accepting new HTTP
connections over the network and placing the descriptor for this connection
into a fixed-size buffer; in your implementation, the main thread
should not read from this connection. The number of elements in the buffer is
also specified on the command line. Note that the existing web server has a
single thread that accepts a connection and then immediately handles the
connection; in your web server, this thread should place the connection
descriptor into a fixed-size buffer and return to accepting more connections.

Each worker thread is able to handle requests. A
worker thread wakes when there is an HTTP request in the queue; when there are
multiple HTTP requests available, which request is handled depends upon the
scheduling policy, described below. Once the worker thread wakes, it performs
the read on the network descriptor, obtains the specified content (by either
reading the static file or executing the CGI process), and then returns the
content to the client by writing to the descriptor. The worker thread then
waits for another HTTP request.

Note that the main thread and the worker threads are in a producer-consumer
relationship and require that their accesses to the shared buffer be
synchronized. Specifically, the main thread must block and wait if the
buffer is full; a worker thread must wait if the buffer is empty. In this
project, you are required to use condition variables. Note: if your
implementation performs any busy-waiting (or spin-waiting) instead, you will
be heavily penalized.

## Part 2: Scheduling Policies

In this project, you will implement one scheduling
policy. Note that when your web server has multiple worker threads running
(the number of which is specified on the command line), you will not have any
control over which thread is actually scheduled at any given time by the
OS. Your role in scheduling is to determine which HTTP request should be
handled by each of the waiting worker threads in your web server.

The scheduling policy is:

- **First-in-First-out (FIFO)**: When a worker thread wakes, it handles the
first request (i.e., the oldest request) in the buffer. Note that the HTTP
requests will not necessarily finish in FIFO order; the order in which the
requests complete will depend upon how the OS schedules the active threads.

## Security

Running a networked server can be dangerous, especially if you are not
careful. Thus, security is something you should consider carefully when
creating a web server. One thing you should always make sure to do is not
leave your server running beyond testing, thus leaving open a potential
backdoor into files in your system.

Your system should also make sure to constrain file requests to stay within
the sub-tree of the file system hierarchy, rooted at the base working
directory that the server starts in. You must take steps to ensure that
pathnames that are passed in do not refer to files outside of this sub-tree. 
One simple (perhaps overly conservative) way to do this is to reject any
pathname with `..` in it, thus avoiding any traversals up the file system
tree. More sophisticated solutions could use `chroot()` or Linux containers,
but perhaps those are beyond the scope of the project.

# Gunrock internals

## Command line arguments
Your C++ program must be invoked exactly as follows:

```bash
$ ./gunrock_web [-p port] [-t threads] [-b buffers]
```

The command line arguments to your web server are to be interpreted as
follows.

- **port**: the port number that the web server should listen on; the basic web
  server already handles this argument. Default: 8080.
- **threads**: the number of worker threads that should be created within the web
  server. Must be a positive integer. Default: 1.
- **buffers**: the number of request connections that can be accepted at one
  time. Must be a positive integer. Note that it is not an error for more or
  less threads to be created than buffers. Default: 1.

For example, you could run your program as:
```
$ ./gunrock_web -p 8003 -t 8 -b 16
```

In this case, your web server will listen to port 8003, create 8 worker threads for
handling HTTP requests, and allocate 16 buffers for connections that are currently
in progress (or waiting).

## Key concepts
The main idea behind this server is to make adding handlers as easy as writing a function. The `FileService.cpp` is a simple service that will read a file from the `static` directory and serve it back to the client as HTML. If you want to write new handlers, you'd do it by adding the new service and inheriting from `HttpService`, adding your source file to the `Makefile` and registering your service with the main `gunrock.cpp` file as a new service.

To match services to requests, the main `gunrock.cpp` logic tries to find the first path prefix match that it can, and when it finds a match it forwards the request on to the service for handling.

From within the service, you set the body of the request, or if there is an error you set the appropriate status code in the response object.

## Thread functions

We created a pthread replacement library, called `dthread`, that you must
use for this project. `dthread` logs information about your use of threads,
mutexes, and condition variables so that we can grade your project.

We anticipate that you will find the following routines useful for creating
and synchronizing threads: `dthread_create()`, `dthread_detach`,
`dthread_mutex_lock()`, `dthread_mutex_unlock()`,
`dthread_cond_wait()`, `dthread_cond_signal()`. To find information on these
library routines, read the man pages for the pthread version of these same
routines. To initialize your mutex and condition variables, assign them
to the `PTHREAD_MUTEX_INITIALIZER` and `PTHREAD_COND_INITIALIZER` macros
and you'll get initialized mutex and conidition variables.

## Key files
To make this server multithreaded, you're going to need to modify the main `gunrock.cpp` file and potentially `FileService.cpp`. You'll need to modify these files so that client requests are handled by a pool of threads with some priority logic to handle high priority files first. See the project README for more details.

## Other files
- **gunrock** - The main function + basic request handling
- **FileService** - Main file service, where the application logic for reading files goes
- **dthread** -- The main threading utilities, use the functions in this file for your threads
- **HTTP** - Higher level HTTP object, interfaces with the `http_parser`
- **http_parser** - HTTP protocol parsing state machine and callback functionality
- **HTTPRequest** - The HTTP request object, this is filled in by the framework
- **HTTPResponse** - The HTTP response object, the data for the response is filled in by the service
- **HttpUtils** - Simple utility functions for working with HTTP data
- **MyServerSocket** - High level abstraction on top of server sockets, accepts connections from new clients
- **MySocket** - High level abstraction on top of sockets, used by the framework to read requests and write responses

