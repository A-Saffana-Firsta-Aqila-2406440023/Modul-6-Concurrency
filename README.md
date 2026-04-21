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
