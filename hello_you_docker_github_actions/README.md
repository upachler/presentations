# Minimum Rust Program -> Docker -> Github Actions

today's goals:
* create a minimal Rust 'hello world' app
* create a Dockerfile for it to dockerize it locally
* put it on github
* configure CI with github actions
* add the docker build step as a new github action


## CREATE PROGRAM

We'll create a little hello world variant:
* prints 'Hello, you' if no arg
* prints 'Hello, [name]' if arg (for [name]) is given

Infos about the Rust Programming Language: https://rust-lang.org


Cargo is to Rust what Maven is to Java...

We now create a new rust project with Cargo:

```bash
cargo new hello_you
```

Let's open the project now and
* view it in VS Code (using the rust-analyser extension)
* have a look at Cargo.toml and main.rs

After that, we run it to see the output:
```bash
cargo run
```

Ok, this prints hello world, but this is wrong - it should greet us!!

Change main.rs into this:
```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    let name_opt = args.get(1);
    let name = match name_opt {
        Some(s) => s,
        None => "you"
    };
    println!("Hello, {}!", name);
}
```

Run it again - without params...
```bash
cargo run
```
... and with param 'guys':
```bash
cargo run guys
```

Also, we don't need cargo to run it - the excutable lives in `target/debug/hello_you`, so we can run it from there, too:

```bash
cd target/debug
./hello_you
./hello_you guys
```

## PUT IT INTO DOCKER
We use Docker Desktop, but there are other ways too. D4D is very convenient, you can get it here: https://www.docker.com/products/docker-desktop

Now we create the Dockerfile in the project:

```docker
FROM ubuntu:18.04

ENV TARGET_DIR=/usr/local/bin
RUN mkdir -p ${TARGET_DIR}
COPY target/*/hello_you ${TARGET_DIR}

ENTRYPOINT [ "/usr/local/bin/hello_you" ]
```

This takes the executable stored in target/debug/hello_you and copies it into a docker image based on ubuntu.
Now build the docker image, and name it 'hello_you' (but we could name it all we want):

```bash
docker build -t hello_you .
```

Let's have a look at the image - see hello_you listed:
```bash
docker image ls | less
```

Now lets run the image:
```bash
docker run --rm hello_you
```

Ugh! This prints:
```
standard_init_linux.go:228: exec user process caused: exec format error
```

What's wrong? Hmm. It's saying something about the executable format... (Note: this will only happen if your PC is on a non-x86 Linux system).

This happens because Rust produces native binaries. These do not run inside a VM (like Java) that abstracts the platform away, but instead, they run directly on the metal (native processor architecture, OS).

So what's the solution?


## USE BUILD CONTAINER 
Yes, Dockerfiles can be used as a build environment! This also has the advantage that we get a pre-configured environment that is not going to change - your own private build slave machine, if you will, that's always properly configured. 

So once again, we start with a Dockerfile:
```docker
FROM rust AS builder

# copy files
COPY . /home/rust/hello_you/
WORKDIR /home/rust/hello_you

# finally, build hello_you!
RUN cargo build

ENTRYPOINT [ "/home/rust/hello_you/target/debug/hello_you" ]
```
This will build our project on top of the 'rust' image (which is some linux with lots of tools) and set our program as entry point.

Build and run:
```bash
docker build -t hello_you .
docker run --rm hello_you
docker run --rm hello_you "guys at almato"
```

So this now all works like we expect :)

Problem: Size!

```bash
docker image ls | less
```
This is _1.25GB_ in size! This is because the 'rust' image is very large by itself. However we can do better. Simply start with a fresh `FROM` command after our actual build:

```docker
FROM rust as builder

# copy files
COPY . /home/rust/hello_you/
WORKDIR /home/rust/hello_you

# finally, build hello_you!
RUN cargo build

FROM ubuntu:18.04

ENV TARGET_DIR=/usr/local/bin
RUN mkdir -p ${TARGET_DIR}
COPY --from=builder /home/rust/hello_you/target/*/hello_you ${TARGET_DIR}

ENTRYPOINT [ "/usr/local/bin/hello_you" ]
```

This we build and run once more:

```bash
docker build -t hello_you .
docker run --rm hello_you
docker run --rm hello_you "dwarves"
```

Now we check the size again:
```bash
docker image ls | less
```
Look, just _66MB_!


## BUILD THE IMAGE ON GITHUB

Init our local repo (if not already initialized).
```bash
git init .
```

Create a repo `hello_you` on our account on http://github.com, and push it up there:
```bash
git remote add origin https://github.com/upachler/hello_you.git
git push
```

Note: To do this on the command line, you'll have to set up a personal access token; github no longer allows username/pw on the command line..


