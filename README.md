This is a list of things I wish I knew sooner about Python and it's popular modules.

# don't use the requests library for request-heavy tasks
The `requests` library always slows down over time. I'd recommend looking into `http.client` for consistent speeds, or `socket`s with ssl wrapping for the absolute best perf., more about this in the next section.

https://github.com/psf/requests/issues/5726

# socket + ssl
Using sockets is probably the most optimized approach for sending HTTP requests in python. It won't bother parsing headers you won't need and it doesn't have 2 layers of HTTP libraries under it like `requests` does. It's great.

```python
import socket
import ssl

# keep this a global in all cases for optimized perf.
ssl_context = ssl.create_default_context()

sock = socket.socket()
sock.settimeout(5)
sock.connect(("www.google.com", 443))

# establish SSL connection
sock = ssl_context.wrap_socket(
    sock,
    server_side=False,
    do_handshake_on_connect=False,
    suppress_ragged_eofs=False,
    server_hostname="www.google.com")
sock.do_handshake()

# send HTTP request
sock.sendall(b"GET /robots.txt HTTP/1.1\r\nHost: www.google.com\r\n\r\n")
# get HTTP response
resp = sock.recv(1024 ** 2)
print(resp)

# close connection
try: sock.shutdown(2)
except OSError: pass
sock.close()
```

In some cases simply following the `Content-Length` header isn't enough, and you'll have to worry about [chunked transfer encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding).

# combine multiprocessing and threading
Threading performance peaks at around `50` threads, at higher amounts you may notice your script becoming jittery and unresponsive. To solve this problem you can use the `multiprocessing` library, which is essentially the same as launching your script multiple times in parallel, but with the benefit of being able to exchange data between running processes (this part is gonna look painful at first sight).

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

# async won't magically make your code faster
https://calpaterson.com/async-python-is-not-faster.html
