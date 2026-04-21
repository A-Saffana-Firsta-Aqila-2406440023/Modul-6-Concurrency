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
