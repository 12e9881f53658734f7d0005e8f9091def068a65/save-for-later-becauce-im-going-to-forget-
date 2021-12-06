This is a list of things I wish I knew sooner about Python and it's popular modules.

Take my advice with a grain of salt, I'm no python master, I'm simply just sharing what I learnt over the past couple of years.

# don't use the requests library for request-heavy tasks
The `requests` library always slows down over time. I'd recommend looking into `http.client` for consistent speeds, or `socket`s with SSL wrapping for the absolute best perf., more about this in the next section.

https://github.com/psf/requests/issues/5726

# sockets + SSL wrapping
Using sockets is probably the most optimized approach for sending HTTP requests in python. It won't bother parsing headers you won't need and it doesn't have 2 layers of HTTP libraries under it like `requests` does. It's great.

```python
import socket
import ssl

blocksize = 1024 ** 2
# use a single SSL context for all conn. for optimized perf.
ssl_context = ssl.create_default_context()

# establish TCP conn. to server
sock = socket.socket()
sock.settimeout(5)
sock.connect(("www.roblox.com", 443))

# upgrade conn. to SSL
sock = ssl_context.wrap_socket(
    sock,
    server_side=False,
    do_handshake_on_connect=False,
    suppress_ragged_eofs=False,
    server_hostname="www.roblox.com")
sock.do_handshake()

# send HTTP requests
for _ in range(3):
    sock.sendall(
        b"GET /robots.txt HTTP/1.1\r\n"
        b"Host: www.roblox.com\r\n"
        b"\r\n")
    # get HTTP response
    response, body = sock \
        .recv(blocksize) \
        .split(b"\r\n\r\n", 1)
    # receive rest of body
    content_length = int(response \
        .split(b"content-length: ", 1)[1] \
        .split(b"\r\n", 1)[0])
    while content_length > len(body):
        body += sock.recv(blocksize)
    print(body)

# close connection
try: sock.shutdown(2)
except OSError: pass
sock.close()
```

In some cases simply following the `Content-Length` header isn't enough, and you'll have to worry about [chunked transfer encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding).

# if you're gonna use requests anyway, for god's sake use Sessions
`requests.get/post` creates a connection then immediately closes it after receiving the response. When you're sending a bunch of requests, a lot of time is wasted on waiting for connections to be established.

When you use a `requests.Session()`, connections are cached for later use:
```python
import requests
import time

for n in range(5):
    raw_start_time = time.perf_counter()
    requests.get("https://www.roblox.com/")
    print(f"{time.perf_counter()-raw_start_time:.2f}s elapsed for requests.get no.{n+1}")

with requests.Session() as session:
    for n in range(5):
        raw_start_time = time.perf_counter()
        session.get("https://www.roblox.com/")
        print(f"{time.perf_counter()-raw_start_time:.2f}s elapsed for requests.Session() no.{n+1}")
```
```
0.22s elapsed for requests.get no.1
0.21s elapsed for requests.get no.2
0.21s elapsed for requests.get no.3
0.21s elapsed for requests.get no.4
0.22s elapsed for requests.get no.5

0.22s elapsed for requests.Session() no.1
0.14s elapsed for requests.Session() no.2
0.14s elapsed for requests.Session() no.3
0.17s elapsed for requests.Session() no.4
0.14s elapsed for requests.Session() no.5
```

# combine multiprocessing and threading
Threading performance peaks at around `50` threads, at higher amounts you may notice your script becoming jittery and unresponsive. To solve this problem you can use the `multiprocessing` library, which is essentially the same as launching your script multiple times in parallel, but with the benefit of being able to exchange data between running processes (this part is gonna look painful at first sight).

Any mutable variables should be defined in `worker_func` or `thread_func` BUT definitely not passed to `worker_func`, unless you're sure it's mutable properties won't be used, or the object is specifically designed to be multiprocessing-compatible (`multiprocessing.Queue`, `multiprocessing.Value`, ..)

```python
import multiprocessing
import threading

THREAD_COUNT = 50
PROCESS_COUNT = 5

def thread_func(counter_value, queue):
    with counter_value.get_lock():
        counter_value.value += 1
        print("total thread count:", counter_value.value)
    queue.put("hello")

def worker_func(counter_value, queue):
    threads = [
        threading.Thread(
            target=thread_func,
            args=(counter_value, queue))
        for _ in range(THREAD_COUNT)]
    for thread in threads: thread.start()
    for thread in threads: thread.join()

# the code below will only be executed on the main process
if __name__ == "__main__":
    queue = multiprocessing.Queue()
    counter_value = multiprocessing.Value("i", 0)
    workers = [
        multiprocessing.Process(
            target=worker_func,
            args=(counter_value, queue))
        for _ in range(PROCESS_COUNT)]
    for worker in workers: worker.start()

    while True:
        msg = queue.get(True)
        print("Received message:", msg)
```

# async won't magically make your code run faster
https://calpaterson.com/async-python-is-not-faster.html
