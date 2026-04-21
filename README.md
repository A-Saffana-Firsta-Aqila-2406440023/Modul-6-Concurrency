# Milestone 1: Single Threaded Web Server

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    println!("Request: {:#?}", http_request);
}
```

The `handle_connection` function takes a `TcpStream` (the live connection to a client) and wraps it in a `BufReader` to enable efficient, line-by-line reading. Using the `.lines()` iterator combined with `.take_while()`, it reads each header line of the incoming HTTP request and stops as soon as it hits a blank line, which marks the end of the headers by HTTP convention. All collected lines are gathered into a `Vec<String>` via `.collect()` and printed to the terminal. At this stage, the function only reads and displays what the client sent, no response is written back.

# Milestone 2: Returning HTML
![Commit 2 Screen Capture](/assets/images/commit2.png)

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

In this updated version of `handle_connection`, the function no longer just reads and prints the request. It now sends a real HTTP response back to the client. It uses `fs::read_to_string` from Rust's standard library to load the contents of `hello.html` from disk into a `String`. The response is then assembled with the `format!` macro, following proper HTTP structure: a status line (`HTTP/1.1 200 OK`), a `Content-Length` header, a blank line separator, and finally the HTML body. Finally, `stream.write_all()` converts the response into bytes and writes them back through the same `TcpStream`, completing the full request-response cycle.

# Milestone 3: Validating Request and Selectively Responding
![Commit 3 Screen Capture](/assets/images/commit3.png)

```rust
{
    ...

    let request_line = buf_reader.lines().next().unwrap().unwrap();

    if request_line == "GET / HTTP/1.1" {
        let status_line = "HTTP/1.1 200 OK";
        let contents = fs::read_to_string("hello.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    }
    else {
        let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    }
}
```

The function now reads only the **first line** of the HTTP request using `.next()`, which contains the method, path, and protocol (e.g. `GET / HTTP/1.1`). It then checks whether that line matches a request for the root path. If it does, the server responds with `200 OK` and `hello.html`. Otherwise, it returns `404 NOT FOUND` and `404.html`. This is how the server decides which response to send based on what the client actually asked for.

```rust
{
    ...

    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response = format!(
        "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
    );

    stream.write_all(response.as_bytes()).unwrap();
}

```

The original `if/else` block duplicated the response-building and sending code in both branches, which is harder to maintain. The refactored version extracts only the parts that differ (the status line and filename) into a single conditional, then runs the shared logic (reading the file, formatting, and writing the response) just once. This follows the DRY (Don't Repeat Yourself) principle and makes future changes easier since there is only one place to update.

# Milestone 4: Simulation Slow Response

```rust
{
    ...

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(10));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    ...
}
```

The `/sleep` route calls `thread::sleep(Duration::from_secs(10))`, which blocks the current thread for 10 seconds before sending any response. Because this server is **single-threaded**, it can only handle one connection at a time. So while it's "sleeping", every other incoming request is stuck waiting. This explains why the browser feels frozen. With many users hitting the server at once, this bottleneck would make it essentially unusable, which is the core reason to eventually move toward a multi-threaded design.

# Milestone 5: Multithreaded Server

A thread pool is a group of pre-spawned threads that sit idle, waiting for work to be assigned to them. Instead of creating a new thread every time a request comes in, the server reuses threads from the pool, keeping the number of active threads fixed and predictable.

Without a thread pool, we have two bad options: handle one request at a time (the server freezes on every slow request), or spawn a brand new thread per request (a 10,000 requests would spawn 10,000 threads, crashing the server). A thread pool gives us the best of both concurrency and a hard cap on resource usage. So the server stays responsive without running out of memory.

When the server starts, it creates a fixed number of Worker threads. In our case, 4. Each worker sits in a loop waiting for a job to arrive through a shared channel. When a new request comes in, `execute()` wraps the work into a Job and sends it into the channel. Whichever worker is free picks it up and runs it. The `Arc<Mutex<...>>` around the channel receiver ensures only one worker grabs each job at a time, preventing race conditions. Once a worker finishes its job, it goes back to waiting and ready for the next one.