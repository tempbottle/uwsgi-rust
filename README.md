# uwsgi-rust
uWSGI and Rust integration plugin


This plugin is an experimental effort to support Rust runtime into uWSGI.

Consider it a proof of concept.

To build (and use) the plugin you need at least a rust-1.x stable environment installed (rustc/cargo binaries must be in the system PATH)

Installation
------------

```sh
uwsgi --build-plugin https://github.com/unbit/uwsgi-rust
```

will generate the rust_plugin.so uWSGI plugin

Or you can make a monolithic build using the network installer (currently probably broken on Linux, works on OSX)

```sh
curl http://uwsgi.it/install | UWSGI_EMBED_PLUGINS="rust=https://github.com/unbit/uwsgi-rust" bash -s nolang /tmp/uwsgi
```

(this will result in the /tmp/uwsgi binary)

Usage
-----

Currently uWSGI expects Rust apps to be managed as shared/dynamic libraries:

```rust
use std::collections::HashMap;

#[no_mangle]
pub extern fn application(environ: HashMap<&str, &str>) -> (String, Vec<(String, String)>, Vec<Vec<u8>>) {

        for (k, v) in environ.iter() {
                println!("{} = {}", *k, *v);
        }

        let mut v = vec![];
        v.push((String::from("Content-Type"), String::from("text/plain")));
        v.push((String::from("Foo"), String::from("Bar")));

        let mut body = vec![];

        body.push(String::from("Hello").into_bytes());

        return (String::from("200 OK"), v, body);
}

```

running

```sh
cargo build --release --manifest-path examples/Cargo.toml
```

will result in `librust_uwsgi_app.so` under `examples/target/release/`

This new library can be loaded in the uWSGI core with the standard `--dlopen` option. Finally you only need to tell uWSGI which rust function to execute ('application' in this case) at every request:

```ini
[uwsgi]
; load the rust plugin (if needed, rememebr you can eventually make monolithic builds)
plugin = rust
; bind to http port 8080
http-socket = :8080
; load the library app
dlopen = ./examples/target/release/librust_uwsgi_app.so
; set 'application' as the entry point
rust-fn = application

; enable master
master = true
; spawn 10 threads in each process/worker
threads = 10
; spawn 8 processes/workers
processes = 8
```

Status
------

prefork and layzy-apps modes are both supported

multiprocessing works

multithreading works (included spawning threads from rust)

Notes
-----

```rust
pub extern fn application(environ: HashMap<&str, &str>) -> (String, Vec<(String, String)>, Vec<Vec<u8>>);
```

is not the final prototype for the entry point (and absolutely not the best one, expecially the `Vec<Vec<u8>>` return value could be very inefficient). Feel free to propose more approaches (we can eventually support multiple of them).

Currently we are still investigating the best approach for accessing the request body (maybe a Reader object could be a good solution).

Monolithic builds on Linux x86_64 looks broken. We are investigating on it.
