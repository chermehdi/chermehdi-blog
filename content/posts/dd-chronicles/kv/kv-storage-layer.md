+++
title = "Distributed systems chronicles: Key value store (3) - Storage layer"
date = 2021-03-27
author = "Mehdi Cheracher"
tags = ["ds-chronicles", "distributed-systems", "architecture", "rust", "storage"]
keywords = ["distributed-systems", "architecture", "key-value-store", "storage"]
description = "Third post of the distributed systems chronicles, creating a storage abstraction, and finishing up a draft version of single node key-value store."
showFullContent = false
+++

## Introduction 

In the previous [post](https://chermehdi.com/posts/dd-chronicles/kv/kv-accepting-requests/), we started accepting client requests, and a minimal set of tests to make sure that everything works. In this post we will draft the Storage layer abstraction, and finish up implementing other commands (`Get`, `Set` and `Clear`). 

## Storage abstraction

This project is meant to describe how a distributed key-value store would work,
along with a series of blog posts to help explain some concepts, that being
said, it's not a production grade storage system, so we are allowed to make some
compromises to keep simplicity and understandability our main goal as opposed to
performance and reliability.

We will try to design a basic Storage abstraction to describe how any Storage
engine for this project would work, and we will provide the simplest possible
implementation to meet our simplicity goals, and also provide the ability of
extension for people wanting to try different things in terms of storage
implementation.


```rust
pub trait Storage {
    fn open(&mut self, dir: String, options: StorageOptions) -> Result<()>;

    fn set(&mut self, key: String, value: String) -> Result<()>;

    fn get(&self, key: &String) -> Result<Option<&String>>;

    fn unset(&mut self, key: &String) -> Result<Option<String>>;

    fn close(self) -> Result<()>;
}
```

The abstraction is provided as a Rust `trait`, which is the Rust way of
describing interfaces like other languages.

The abstraction provides a couple of methods that are part of the lifecycle of
a storage engine `open` and `close`, and this gives hooks for initialisation
work (opening file handles, creating Data structures necessary for correct
behaviour ...), and cleanup (close resources, flush data to disk / network ...).

This abstraction provides the set of methods that you expect to find in a key
value store: `get`, `set` and `unset`.

Below is the simplest possible in-memory implementation:

```rust
pub struct InMemStorage {
    db: HashMap<String, String>,
}

impl Storage for InMemStorage {
    fn open(&mut self, _dir: String, _options: StorageOptions) -> Result<()> {
        Ok(())
    }

    fn set(&mut self, key: String, value: String) -> Result<()> {
        self.db.insert(key, value);
        Ok(())
    }

    fn get(&self, key: &String) -> Result<Option<&String>> {
        Ok(self.db.get(key))
    }

    fn unset(&mut self, key: &String) -> Result<Option<String>> {
        Ok(self.db.remove(key))
    }

    fn close(self) -> Result<()> {
        Ok(())
    }
}
```

As you can see, the trait implementation just delegates to
a `std::collections::HashMap` to keep key-value pairs in memory, people can
experiment with other more sophisticated implementations ... 

Now our storage interface looks like this:

```rust
type StorageEngine = Arc<Mutex<Box<dyn Storage + Send + Sync>>>;
```

The type definition is quite verbose, but it's not for nothing:

- The storage engine should be put in a `Box` which means that it's a heap
  allocated object, as traits don't have a size known at compile time, so rust
  does not know how much space it will need on the stack, thus the need for heap
  allocation.
- The object in the box should be transferable across thread boundaries thus
  the need for `Send` and `Sync`.
- The object should be safe to be accessed from multiple threads, thus the need
  for a `Mutex`.
- The object should be thread-safe copyable, thus the need of an Atomically safe
  reference counted pointer, or `Arc`.

To not have to specify this long type signature everywhere we need to use the
storage, we created a type alias called `StorageEngine` to be used instead.

We also introduced a new type called `Executor` that keeps a reference of
a storage engine and a handler, and it has a `1:1` relationship with the client,
and it will be responsible for handling client requests:


```rust
pub(crate) struct Executor {
    handler: ConnectionHandler,
    store: Arc<Mutex<Box<dyn Storage + Send + Sync>>>,
}
```

At creation time, the `Executor` takes ownership of a `StorageEngine` instance and
a `ClientHandler` and keeps running and executing commands until and error
happens (the command sent by the client cannot be parsed), or the client closes
the connection.

```rust
impl Executor {
    pub(crate) fn new(handler: ConnectionHandler, store: StorageEngine) -> Self {
        return Executor { handler, store };
    }

    pub(crate) async fn run(&mut self) -> Result<()> {
        loop {
            let cmd = match self.handler.read_command().await {
                Ok(val) => val,
                Err(_msg) => None,
            };
            if let Some(cmd) = cmd {
                execute_cmd(&mut self.store, &mut self.handler, cmd).await?;
            } else {
                return Err("Connection closed / Poisoned message".into());
            }
        }
    }
}
```

To execute a command, we only need a reference to the `StorageEngine`,
a `ConnectionHandler` to write the response to and the `Command` to execute, the
`execute_cmd` function would match on the command type, and delegate to the
correct handler method: 

```rust
async fn execute_cmd(
    store: &mut StorageEngine,
    handler: &mut ConnectionHandler,
    cmd: Command,
) -> Result<()> {
    let result = match cmd {
        Command::Ping(key) => handle_ping(key),
        Command::Set(key, value) => handle_set(store, key, value),
        Command::Get(key) => handle_get(store, key),
        Command::Clear(key) => handle_unset(store, key),
    };
    handler.write_response(&result).await?;
    Ok(())
}
```

Changing the underlying `StorageEngine` requires taking a lock as it's guarded
by a `Mutex` as described above, as the same instance would be accessible from
multiple threads.

Taking a `Mutex` lock will return a `MutexGuard` type (if the lock could be
obtained), which can be used to change the underlying resource, once the guard
is out of scope, it will be destroyed and the lock will be released. 

```rust
fn handle_set(store: &mut StorageEngine, key: String, value: String) -> Response {
    let mut guard = store.lock().unwrap();
    return match guard.set(key.clone(), value) {
        Ok(_) => Response::Ok(key),
        Err(_) => Response::Error(String::from("Error happened while setting the key")),
    };
}
```

The set of tests to make sure that this operation works is going to be written
in the same way we did for the `Ping` command in the last post, let's look at
a **Set then Override** test:

```rust
#[tokio::test]
async fn test_set_override() {
    let addr = start_server().await.unwrap();
    let mut client = kvstore::client::create(addr).await.unwrap();
    let res = client
        .set(String::from("key"), String::from("value1"))
        .await
        .unwrap();
    assert_eq!(res, Some(Response::Ok(String::from("key"))));

    let res = client
        .set(String::from("key"), String::from("value2"))
        .await
        .unwrap();
    assert_eq!(res, Some(Response::Ok(String::from("key"))));

    let res = client.get(String::from("key")).await.unwrap();
    assert_eq!(res, Some(Response::Ok(String::from("value2"))));
}
```

The client sets a `key` with value `value1` and then it overrides it with
`value2` then queries the `key` to make sure that the last value is the one
that is stored on the server.


## Conclusion

This post was somewhat purely technical, although it didn't contain anything
specific to distributed systems, it was necessary to describe the current state
of the project and rational behind some code changes.

Now that we have a single node key-value store, how can we make it distributed,
The following posts will try to describe how to do just that.

For the rest of the code, you can find it [here](https://github.com/chermehdi/ds-chronicles).
