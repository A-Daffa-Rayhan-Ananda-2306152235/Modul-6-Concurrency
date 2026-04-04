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