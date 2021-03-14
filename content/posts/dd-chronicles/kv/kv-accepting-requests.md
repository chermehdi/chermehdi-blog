++
title = "Distributed systems chronicles: Key value store (2) - Accepting requests"
date = 2021-03-14
author = "Mehdi Cheracher"
tags = ["ds-chronicles", "distributed-systems", "architecture", "rust"]
keywords = ["distributed-systems", "architecture", "key-value-store"]
description = "Second post of the distributed systems chronicles, Now that we have a protocol in place, let's start accepting some client requests"
showFullContent = false
+++

## Introduction 

In the previous [post](https://chermehdi.com/posts/dd-chronicles/kv/kv-architecture-protocol), We designed a communication protocol to be used between the client and the key-value store server. In this post, we will start accepting requests, and test if our protocol works as expected. 

## Accepting requests 

To start accepting commands, we will need some form of a transport to carry the
bytes representing our commands, for that we will start a TCP server powered by
the `tokio` library.

The server will handle accepting requests, and spawning a separate `tokio` task
per client connection to handle command parsing and execution.

```rust
pub async fn run(listener: TcpListener) {
    loop {
        let (stream, _) = listener.accept().await.unwrap();

        // start a new tokio worker task to handle client connections.
        tokio::spawn(async move {
            if let Err(msg) = process(stream).await {
                println!("An error happened while processing the request: {:?}", msg);
            }
        });
    }
}
```

The code for the `run` function is straight forward, given a `TcpListener`, which
we will create at startup based on some parameters given by the user, start
accepting clients connections, and handle each connection in it's own separate
`tokio` task, this will allow us to not block the main server thread, and scale our
workload accross multiple CPU cores thanks to the `tokio` runtime task scheduler.

```rust
async fn process(stream: TcpStream) -> Result<()> {
    let mut handler = ConnectionHandler::new(stream);
    loop {
        match handler.read_command().await {
            Ok(val) => match val {
                Some(cmd) => {
                    handler.execute(cmd).await?;
                }
                None => {
                    break;
                }
            },
            Err(msg) => {
                return Err(msg);
            }
        }
    }
    Ok(())
}
```

The `process` function takes ownership of the `TcpStream`, and will delegate the
command parsing and execution to another component that is created per
connection: `ConnectionHandler`.

The function will try to continuously read commands from the client and execute
them as they come, and it will stop once the connection is closed from the
client, or an error happens.

```rust
pub struct ConnectionHandler {
    stream: BufWriter<TcpStream>,

    buf: BytesMut,
}
```

The `ConnectionHandler` type keeps a reference of a **buffered** `TcpStream`, and
an internal buffer to read bytes coming from the TCP stream.

Buffering the `TcpStream` is necessary to not cause individual write calls to
issue individual syscalls which is expensive and unnecessary. instead, we wan't the stream to only issue the write
syscalls once the buffer is full or we explicitly flush it.

The read buffer heald by the `ConnectionHandler` is a type from the `bytes`
crate, and you can look at it as a `[u8]` but with a lot of convinience methods,
it is created with a fixed capacity (defaulted to `1024`), i.e we suppose that each command shouldn't
take more than a given size, and if that isn't the case, we will deal with it
later on.

Reading the command from the stream is as simple as calling `protocol::Parser::parse`
from the previous blog post, but before doing so we must ensure that the buffer
is full or has some data, this is done before every call to `read_command` by
calling the `ensure_filled` function.

```rust
pub async fn read_command(&mut self) -> Result<Option<Command>> {
    if let None = self.ensure_filled().await? {
        // `None` is returned when the connection is closed and no bytes can be read.
        return Ok(None);
    }

    let mut cursor = Cursor::new(&self.buf[..]);
    let command = Parser::parse(&mut cursor)?;
    let length = cursor.position() as usize;
    self.buf.advance(length);
    Ok(Some(command))
}
```

When trying to parse a command from a buffer, we are always sure that the buffer
has some data  and the cursor to the buffer is always moved after
a parse succeeds.

> **Note**: in a future post we will add a pre-scanning for the contents
  of the buffer, to make sure that a command parse attempt will most likely succeed, and
  if we detect invalid or unexpected bytes, we error-out before doing any more
  unnecessary work which could improve performance.)

After execting a command the response is written by calling the
`protocol::Writer::write` from the previous post as well.

For this post we only have one one single command, which a test command to
verify that the server is responsive, the `ping` command.

```rust
pub async fn execute(&mut self, cmd: Command) -> Result<()> {
    match cmd {
        Command::Ping(key) => {
            return self.handle_ping(key).await;
        }
        _ => {
            // TODO: future posts.
            unimplemented!();
        }
    }
}

async fn handle_ping(&mut self, key: String) -> Result<()> {
    if key.is_empty() {
        // Default to a ping.
        self.write_response(&Response::Ok("PONG".into())).await?;
    } else {
        self.write_response(&Response::Ok(key)).await?;
    }
    Ok(())
}
```

A `ping` command will just return `PONG` _(the way `Redis` does)_, or it will return the
passed string.

To make sure that our whole setup works, we will create an integration test,
where we will start a server on a random port, and we will execute the ping
command and make sure that the response is what we expect.

To be able to do this, we should have a simple client to make interacting with
the server easier, a simple client could look like this:

```rust
pub struct Client {
    handler: ConnectionHandler,
}

impl Client {
    pub async fn ping(&mut self, key: String) -> Result<Option<Response>> {
        let command = Command::Ping(key);
        self.handler.write_command(&command).await?;
        return self.handler.read_response().await;
    }
}
```

Now our tests will live under the `kvstore/tests` directory, and the ping test
will look something like:

```rust
async fn start_server() -> Result<SocketAddr> {
    let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(async move { kvstore::server::run(listener).await });
    Ok(addr)
}

#[tokio::test]
async fn test_ping() {
    let addr = start_server().await.unwrap();
    let mut client = kvstore::client::create(addr).await.unwrap();
    let res = client.ping(String::from("")).await.unwrap();
    assert_eq!(res, Some(Response::Ok(String::from("PONG"))));
}

#[tokio::test]
async fn test_ping_with_value() {
    let addr = start_server().await.unwrap();
    let mut client = kvstore::client::create(addr).await.unwrap();
    let res = client.ping(String::from("Value")).await.unwrap();
    assert_eq!(res, Some(Response::Ok(String::from("Value"))));
}
```

`start_server` will start a kvstore server on a random unused port, and will
return the address to the created server, which we will use to
connect our client and start issuing commands by creating the corresponding
objects from the protocol module and trying to `assert` on the returned values.

## Conclusion

In this post we successfully managed to create a TCP server and started
accepting connections and executing `ping` commands, in the following posts we
will dig into the data model, and how we will persist the data in disk/in-memory
and we will implement `get` and `set` commands.

The full source code could be found [here](https://github.com/chermehdi/ds-chronicles), stay tunned!
