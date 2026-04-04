## Reflections

### Reflection 1

In milestone 1, we implemented a single-threaded web server that listens for incoming TCP connections and reads the HTTP request from the browser. The core of this process happens in the `handle_connection` method. Here is a breakdown of what the code does:

* **`mut stream: TcpStream`**: The method takes a mutable reference to a `TcpStream`. The stream represents a connection between the client (browser) and the server. It needs to be mutable (`mut`) because reading from the stream changes its internal state (it keeps track of what data has already been read).
* **`BufReader::new(&mut stream)`**: Instead of reading directly from the stream byte-by-byte (which is slow and requires many system calls), we wrap the stream in a `BufReader`. This buffers the reads, fetching chunks of data at a time, making network operations much more efficient.
* **`.lines()`**: This method creates an iterator over the data in the `BufReader`, yielding it line by line. It automatically splits the incoming data stream whenever it sees a newline character.
* **`.map(|result| result.unwrap())`**: Reading from a network stream can fail, so `lines()` yields `Result<String, std::io::Error>`. The `map` function applies a closure to each item. By using `unwrap()`, we are telling Rust to extract the `String` if successful, or immediately stop the program (panic) if an error occurs.
* **`.take_while(|line| !line.is_empty())`**: In the HTTP protocol, the request headers are separated from the body by an empty line. This iterator method continuously takes lines from the stream *until* it encounters an empty line, effectively capturing only the HTTP request headers and the request line.
* **`.collect()`**: This takes all the yielded string items from the iterator chain and collects them into a data structure. We specified the type `Vec<_>`, so Rust infers it should collect these lines into a Vector (a growable array) of Strings. 

Finally, `println!("{:#?}", http_request);` takes that vector and pretty-prints it to the console, allowing us to inspect the raw HTTP request sent by the browser.

### Reflection 2

![Commit 2 screen capture](/assets/images/commit2.png)

In this milestone, we upgraded our server from simply reading a request to actually sending back a valid HTTP response containing an HTML page. Here is a breakdown of the new code added to `handle_connection`:

* **`let status_line = "HTTP/1.1 200 OK";`**: This defines the standard HTTP success response. It tells the browser that the request was received, understood, and accepted.
* **`let contents = fs::read_to_string("hello.html").unwrap();`**: We use Rust's standard filesystem module (`std::fs`) to read the entire contents of our `hello.html` file into a string. The `unwrap()` method handles potential errors (like if the file is missing) by panicking and stopping the program.
* **`let length = contents.len();`**: We calculate the size of our HTML payload. This is crucial for the HTTP headers.
* **`let response = format!(...);`**: Here we construct the raw HTTP response exactly as the protocol requires:
    * First, we send the `status_line`.
    * Next, we send the `Content-Length` header. This tells the browser exactly how many bytes of data to expect in the body, ensuring it knows when the response is complete.
    * We use `\r\n` (carriage return and line feed) to separate lines, which is the standard line ending for HTTP headers.
    * Crucially, we use `\r\n\r\n` to create a blank line. In the HTTP protocol, this blank line is the signal to the browser that the headers are finished and the actual body (our HTML `contents`) is about to begin.
* **`stream.write_all(response.as_bytes()).unwrap();`**: Finally, we convert our perfectly formatted `response` string into raw bytes and write them directly back into the TCP stream. This sends the data over the network to the waiting browser.

### Reflection 3

![Commit 3 screen capture](/assets/images/commit3.png)

In milestone 3, we upgraded the server from returning the same HTML page for every request to selectively responding based on the requested URL path. We also applied some basic code refactoring. Here is a breakdown of what was done and why:

**1. Splitting the Response (Validating Requests)**

To respond differently to different requests, we first needed to isolate the exact request line (e.g., `GET / HTTP/1.1`). 
* We used `buf_reader.lines().next().unwrap().unwrap()` to grab just the first line of the HTTP request.
* We then introduced an `if / else` block. If the `request_line` exactly matches `"GET / HTTP/1.1"` (meaning the user is asking for the root of the website), we serve them `hello.html` with a `200 OK` status. 
* If they request *anything else* (triggering the `else` block), we serve a newly created `404.html` error page alongside a `404 NOT FOUND` status line.

**2. Why the Refactoring is Needed**

Initially, writing out the `if / else` blocks resulted in a lot of duplicated code. Both blocks contained the exact same logic for reading a file to a string, calculating its length, formatting the HTTP response, and writing it back to the TCP stream. The *only* things that actually differed were the `status_line` and the `filename`.

Refactoring is necessary here to adhere to the **DRY (Don't Repeat Yourself)** principle. By pulling the differing values into a tuple:
```rust
let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
    ("HTTP/1.1 200 OK", "hello.html")
} else {
    ("HTTP/1.1 404 NOT FOUND", "404.html")
};
```
...we can run the file-reading and response-writing code unconditionally just *once* at the end of the method. 

**The main benefits of this refactoring are:**
* **Conciseness:** The code is much shorter and easier to read.
* **Maintainability:** If we ever need to change how the server reads files or formats the final HTTP response string, we only have to update the code in *one* place, rather than updating it inside every single `if/else` branch we create in the future.