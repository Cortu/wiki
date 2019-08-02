# Networking using `asyncio`

> Level: Advanced

## Event-loops vs Threads

The Python `asyncio` module proposes an event-loop implementation and utilities for I/O operations.

Event-loops were designed to address a simple issue: In some cases, your program is mostly waiting for I/O operations to complete.

This means that if you read from a file, or make an http request, or anything that requires some time, your thread will block until your get a response.

Event-loops solve that issue by making you write your program in such a way that you can execute something else while requests are pending.  This alows you to run a process across multiple tics<sup>According to prototyping.</sup>.

But this is not magic, in order for an event-loop to work and "parallelize" your code, it needs to "take control of your thread".

A program only ever has one thread of execution by default, you can spawn more threads but there are some issues:

- It is relatively heavy to create a new thread from Python.

  _Spawning threads require some processing from the runtime and the OS._

- You cannot expect to spawn 1 thread per task you want to complete.

  _If you are parallelizing 5 tasks, why not. But if you have 100+ tasks, threads are not the way to go anymore. You can use pools, where you define a maximum limit of threads that can run at the same time, then "submit" a function to the pool and it will try to find available threads to run the submitted function. Problem is that if all the threads are busy, new work won't happen._

- Code actually runs in parallel.

  _While this might be interesting to "speed up" your processing, it is also very easy to hit problems if you access the same resources from different threads. It is especially an issue in the BGE, since there are specific times at which the BGE is "ready" to execute logic code, but if your run functions in separate threads, you might call APIs outside of these "windows" and the BGE might crash or raise an exception in your code running in the thread._

Threads are not inherently bad, but they must be used with a lot of care, especially in the BGE.

So how can an event-loop help you?

First of all, it can run on just one thread, but allow you to start multiple operations at the same time, without blocking waiting for an answer.

Let's take a look at how a synchronous code may look like (script for a Python Controller in "module" mode):

```py
# Echo client program
# https://docs.python.org/3/library/socket.html
import socket

HOST = 'daring.cwi.nl'    # The remote host
PORT = 50007              # The same port as used by the server

def pycontroller_on_event(controller):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
        client_socket.connect((HOST, PORT))
        client_socket.sendall(b'Hello, world')
        data = client_socket.recv(1024)

    print('Received', repr(data))
```

This code looks fairly simple, it basically creates a socket, sends a message (`.sendall`), and finally waits for a response (`.recv`).

The issue here is that the whole program is blocked on the receive statement. Because of the way it is written, you don't want your program to continue if you did not receive anything, that wouldn't make any sense. So it blocks.

That's the price of writing code for only one thread, if it blocks, you'll have to wait.

In the BGE this would translate into your game freezing, because it will wait for your script to finish, which itself waits for a response.

To solve this issue, you have three solutions:

1. Make the socket non-blocking.

   This will make the `.recv` call return and raise an error if there was nothing to read.
   You need to write your code to branch off and wait for the next execution to try and read again, until something can be read.
   It works, but throwing errors on each logic frame and re-executing the same code over and over until it works is a bit annoying to reason about.

   ```py
   # <sub>TCP over a WAN will almost certainly take longer than one tic.  This will only work in a LAN.  It almost certainly will block while sending or receving a packet.</sub>
   # Echo client program (non-blocking)
   # https://docs.python.org/3/library/socket.html
   import socket

   HOST = 'daring.cwi.nl'    # The remote host
   PORT = 50007              # The same port as used by the server

   def pycontroller_on_event(controller):
       owner = controller.owner

       client_socket = owner.get('socket', None)

       # Test if someone already expects a response somehow
       if client_socket is None:
            # <sub>Using two equation signs in one command is extreemly uncommin.  This needs an explanation, even in an advanced tutorial.</sub>
            client_socket = owner['socket'] = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            # <sub>Was s defined somewhere?  I appologise if I'm just not seeing it.</sub>
            s.settimeout(0)
            s.connect((HOST, PORT))
            s.sendall(b'Hello, world')

    def pycontroller_always(controller):
        owner = controller.owner

        client_socket = owner.get('socket', None)

        # Test if someone somehow expects a response
        if client_socket is not None:
           try:
               data = s.recv(1024)
           except socket.timeout:
               pass

           if data is not None: # finally got something to read
               print('Received', repr(data))
               client_socket.close()
               del owner['socket']
   ```

   In this version, we are constantly "polling" on a socket (if present) in order to see if we get a response. In order to do this, we need to store our "useful state" inside the game object that executes the code, and the way it is written here, we can only have one pending request at the time. Allowing for multiple request is left to the reader.

   The main conclusion I have from this is that while it works, it requires to store the state "outside" of your functions, which can make it harder to design bigger systems. The polling is also a bit ugly, even though this is a naive implementation, supporting more advanced use cases would be even funnier.

2. Read in a different thread.

   This way, your main thread will finish, but the new one will wait until something is received.
   The issue here is that you don't know when it will finish, and it might be at a time the BGE is not ready.

   ```py
   # Echo client program (threading)
   # https://docs.python.org/3/library/threading.html
   import threading
   # https://docs.python.org/3/library/socket.html
   import socket

   HOST = 'daring.cwi.nl'    # The remote host
   PORT = 50007              # The same port as used by the server

   def recv_thread(sock):
       '''
       This will be run in its own thread
       '''
       data = sock.recv(1024)
       print('Received', repr(data))
       sock.close()

   def pycontroller_on_event(controller):
       client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM):
       client_socket.connect((HOST, PORT))
       client_socket.sendall(b'Hello, world')
       thread = threading.Thread(target=recv_thread, args=(client_socket,))
       thread.start()
   ```

   In this scenario, the code looks a bit cleaner, but this is only because this is a simple example.
   Once you start doing more complex things, especially upon reception, you will start enjoying using threads.
   If you try to use a gameobject reference inside the thread, using the BGE API will only bring problems.

3. Use `asyncio`.

   ```py
   # python >= 3.6
   # https://docs.python.org/3/library/asyncio.html
   import asyncio

   async def tcp_echo_client(message):
       reader, writer = await asyncio.open_connection(
           '127.0.0.1', 8888)

       print(f'Send: {message!r}')
       writer.write(message.encode())

       data = await reader.read(100)
       print(f'Received: {data.decode()!r}')

       print('Close the connection')
       writer.close()
       await writer.wait_closed()

   def pycontroller_on_event(controller):
       asyncio.create_task(tcp_echo_client('Hello World!'))

   def pycontroller_always(controller):
       logic_frame_time = 1 / bge.logic.getLogicTicRate()
       # <sub>Why not just use asyncio.sleep(0)?  This waist logic time for no apperant reason.</sub>
       asyncio.run(asyncio.sleep(logic_frame_time / 3)) # arbitrary number
   ```

   Or for older Python runtimes:

   ```py
   # python < 3.6
   # https://docs.python.org/3/library/asyncio.html
   import asyncio

   @asyncio.coroutine
   def tcp_echo_client(message, loop):
       reader, writer = yield from asyncio.open_connection(
           '127.0.0.1', 8888, loop=loop)

       print('Send: %r' % message)
       writer.write(message.encode())

       data = yield from reader.read(100)
       print('Received: %r' % data.decode())

       print('Close the socket')
       writer.close()

   def pycontroller_on_event(controller):
       loop = asyncio.get_event_loop()
       loop.ensure_future(tcp_echo_client('Hello World!', loop))

   def pycontroller_always(controller):
       logic_frame_time = 1 / bge.logic.getLogicTicRate()
       loop = asyncio.get_event_loop()
       loop.run_until_complete(asyncio.sleep(logic_frame_time / 3)) # arbitrary number
   ```

   Please note how recent version are much prettier!

   What happens here looks a bit complex, but it is also very powerful. We run an event-loop during 1/3 of a logic frame time <sub>I'm pretty sure the event-loop runs even when your not sleeping.  Try runing it with asyncio.sleep(0)</sub> (roughly 16.7ms for a logic frame, which means 5.57ms for asyncio here). During this time, the event-loop will block the whole thread and do its thing, waiting for I/O events to happen in order to resume registered tasks.

   In places where you did use the `await` keyword, your code can be put on "wait" (as in, not executed anymore) without blocking the whole thread. The execution state is kept in memory, but something else may execute in place. Once whatever that was awaited returns, the event-loop may resume execution of the coroutine that was stuck waiting for the event, until the next `await` or `return` statements.

   In Python, coroutine execution is wrapped by a `Task` object, which shares ideas with `Future` objects.

   They <sub>they who?</sub> basically represent the state of the execution <sub>the execution of what?</sub> : from `pending` to `cancelled`, `rejected` or `resolved`<sub>How are these states retreved?</sub>. Since a coroutine function can be put on hold depending on the I/O state <sub>How?</sub>, it makes sense to get a return value <sub>Are you saying that you are getting a return value from some where, or are making one of the examples return something?</sub> that represents this process of resolution <sub>are you refering to the coroutines resolution or the example fucntions?</sub>, which is anything but instantaneous.  <sub>I feel that this last sentince is confusing and says very little.  It should probly be acompanied by code examples, removed, or converted to a stub indicating the need for future eludiation.</sub>
