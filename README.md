# How to compile Rust for a new OS
**_A collaborative tutorial_**

As my Bachelor thesis, I'm working on a [HelenOS](https://www.helenos.org/) target for Rust. Configuring the Rust compiler is a complex task and I was often stuck without any idea what to do. Eventually, I will need to write some text of my thesis which could also serve useful, but 

### Get involved!

**Are you also trying to build Rust programs for an OS unsupported by Rust?**

Great! Don't be afraid to raise issues with questions. Maybe I ran into the same issue, and just missed it in the tutorial. Or maybe we'll figure it out together. After that, we can update the tutorial text with a summary.

**Did you deal with some issue not covered here?**

Congrats! Please, submit a PR to retain this knowledge for future ~~generations~~ developers.

**Did you already go through this process, and want to help others?**

Amazing! Please enable _Watch_ on this repository. This will send you notifications about new issues.

## The tutorial

No more talking. Here we go.

### 1. Avoid compiling Rust compiler for as long as possible

As a first step, create a custom `target.json` file and create a minimal `no_std`, `no_main` project. Your goal now show be to create a _Hello world!_ executable, that you can successfully run on your OS.

Create a new cargo project, add any linking configuration to `build.rs`, and try to link against your OS's libc (statically, later optionally dynamically). Write a simple C-like main, which calls `unsafe { puts() }` or something similar.


<details>
<summary>How to get some <code>target.json</code></summary>

Choose an existing target similar to what you're building for (mainly same CPU architecture, and something quite basic - probably not Linux, which likely has many fancy extra features enabled). See [list of Rust targets](https://doc.rust-lang.org/rustc/platform-support.html), or `rustc --print target-list`.

Get the JSON for the existing target with `rustc +nightly -Z unstable-options --print target-spec-json --target i686-unknown-helenos` (use the target that you selected). You will need to patch a few things - you probably want to use your own linker, and you want to change the OS in `os`, `metadata`, `llvm-target` etc. (although I'm not sure what effect it has, HelenOS is not in llvm and it works)

Here's what target.json looks like for my simple OS - I'm not sure I have everything right, but it works.

```
{
  "arch": "x86",
  "cpu": "pentium4",
  "crt-objects-fallback": "false",
  "crt-static-allows-dylibs": true,
  "crt-static-default": true,
  "data-layout": "e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-i128:128-f64:32:64-f80:32-n8:16:32-S128",
  "dynamic-linking": true,
  "has-rpath": true,
  "linker": "i686-helenos-gcc",
  "linker-flavor": "gnu-cc",
  "llvm-target": "i686-unknown-helenos",
  "max-atomic-width": 64,
  "metadata": {
    "description": "IA-32 (i686) HelenOS",
    "host_tools": false,
    "std": true,
    "tier": 3
  },
  "os": "helenos",
  "panic-strategy": "abort",
  "plt-by-default": false,
  "position-independent-executables": true,
  "pre-link-args": {
    "gnu-cc": [
      "-m32"
    ],
    "gnu-lld-cc": [
      "-m32"
    ]
  },
  "relro-level": "full",
  "stack-probes": {
    "kind": "inline"
  },
  "static-position-independent-executables": true,
  "target-pointer-width": "32"
}
```
</details>

<details>
<summary>Example minimal project to get running</summary>

`src/main.rs`

```
#![no_std]
#![feature(core_intrinsics)]
#![no_main]

use core::{ffi, intrinsics::abort};

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    abort()
}

extern "C" {
    pub fn puts(c: *const ffi::c_char);
}

#[no_mangle]
fn main(argc: ffi::c_int, argv: *const *const ffi::c_char) -> ffi::c_int {
    let str = b"Hello World!\0".as_ptr() as *const ffi::c_char;
    unsafe {
        puts(str);
    }
}
```

`build.rs`
```
fn main() {
    println!("cargo:rustc-link-search=native=../../helenos/ia32dyn/export-dev/lib");
    println!("cargo:rustc-link-lib=static=startfiles"); // some C runtime (symbols like `_start`)
    println!("cargo:rustc-link-arg=-nostartfiles"); // IIRC I had this to avoid gcc including some default runtime for linux, which obviously didn't work. You might need different link args

    // libc:
    // static:
    println!("cargo:rustc-link-lib=static=c");

    // or dynamic:

    // ideally this would be
    // println!("cargo:rustc-link-lib=dylib=c");
    // but for some reason rustc decides that it likes the static version better, so I had to use this:
    // println!("cargo:rustc-link-arg=-l:libc.so.0");
}
```

`.cargo/config.toml`

```
[build]
target = "../path/to/your/target.json"

[unstable]
build-std = ["core"]
```
</details>

Now you can say that Rust is already running on your OS. I can hear you say, _"well, it's just a glorified wrapper around raw, unsafe libc APIs"_. That is true. But now we know that running Rust on your OS is possible, and that's important!

### 2. Get `alloc` working

Add your libc `malloc` or equivalent to the `extern "C"` block of definitions and implement `GlobalAlloc` (see code below). Add `"alloc"` to the list of `build-std` crates, and try to use a `alloc::vec::Vec`.


<details>
<summary>Code needed for <code>alloc</code></summary>

```
struct System;

#[global_allocator]
static A: System = System;

unsafe impl GlobalAlloc for System {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 { /* TODO */ }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) { /* TODO */ }
}
```

see https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html
</details>

Are you running programs that can use vectors, btreemaps, linked lists, strings, reference-counted objects? That's a big win. You can now run safe, performant algorithms!

### 3. Now comes the hard part - configuring Rust compiler

This part is messy. You are working across multiple git repositories (Rust compiler, libc and the application that you are trying to run), and making isolated commits is hard.

Better documentation of this part is very much TODO (I'm still not done myself). You can reference my current patches here:

- [rust](https://gitlab.mff.cuni.cz/teaching/nprg045/horky/volfmat1/rust/-/compare/master...wip-rust-helenos)
- [libc](https://gitlab.mff.cuni.cz/teaching/nprg045/horky/volfmat1/rust-libc/-/compare/0.2.169...helenos)
- [cc-rs](https://github.com/mvolfik/cc-rs/compare/main...mvolfik:cc-rs:wip-rust-helenos)

#### 3.1 Setup

- Clone locally `rust-lang/rust`, `rust-lang/libc` and `rust-lang/cc-rs`.

- In `cc-rs`, you just need to add the target triplet with your OS as "known", manually to the generated file. This is an unfortunate step, not sure how to avoid it. Then link `cc-rs` in `rust/src/bootstrap/Cargo.toml` (see my diff of rust).

- `libc` is the standard place to list all the Rust definitions of C APIs provided by your OS's libc. Then you need to link to this locally patched libc in `rust/library/Cargo.toml`. For now, just create the respective files and add the module definition, you can leave `libc` definitions empty, and just add things as you go, as you will need them.

- In `rust/compiler/rustc_target`, you now need to add the definition of the target as a "compiler-native" one. This will mostly replicate the `target.json` file you created earlier. **I don't recommend marking your OS as unix-family from the start, even if it is.** There is a big amount of APIs that are enabled by default for `cfg(unix)`, which will make your life hard at the start.

- Create `config.toml` - run `./x setup`, which will guide you through the process. Here's what I have:

```
profile = "library"
change-id = 134650

[rust]
deny-warnings = false
optimize = 2
```

#### 3.2 Your first build

- Now you can attempt first build of the standard library. `./x build library --stage 1 --target i686-unknown-helenos,x86_64-unknown-linux-gnu` (you will need your new target, but also the target of the OS that you're working on). You will get some errors about missing functions, but there shouldn't be too many of them.

  - thread-local storage: this is quite magical, and I uncovered some bugs in my target OS when working on this. Using existing Unix pthread implementation is quite easy, if you have it in your OS.
  - allocator: you have already done this in part 2! If you have `malloc`, you can probably just add the symbols to `libc`, and enable the existing implementation in libstd.
  - ?? what else was here now? I don't remember.

Once you have your compiler configured, and only make changes in `library/std/**` to support your OS, add `--keep-stage-std 0` to the build command. Your changes to the `std` only affect the new OS, so you don't need to rebuild stage0 std and the stage1 compiler, since they are both for your host OS.  
See the documentation of bootstrapping: https://rustc-dev-guide.rust-lang.org/building/bootstrapping/what-bootstrapping-does.html

#### 3.3 Using your build libstd in an application

In rust source: `rustup toolchain link dev-stage1 build/host/stage1`. In app: `rustup override set dev-stage1`.

TODO: more about linking?

### 4. More complete Rust support

- enable and implement all the libstd APIs (filesystem, networking, threads...)
- enable panic unwinding (not just `panic=abort`)
- patch all the crates with OS-specific code
  - if you try to compile basically any existing Rust project for your OS, you will likely run into this :(

### 5. Final boss: Rust compiler inside your OS

???

### Useful links

TODO: clean this up. Some of it is only relevant for HelenOS.

- Slides from NSWI015 https://github.com/devnull-cz/unix-linux-prog-in-c/releases/tag/v99
- https://users.rust-lang.org/t/how-to-cross-compile-rust-program-for-non-linux-os-with-gcc-toolchain-for-aarch64/94835
- A lot of HelenOS wiki (just open the index page and read whatever title sounds vaguely relevant to what you are doing)
- https://os.phil-opp.com/freestanding-rust-binary/#the-eh-personality-language-item
- Inspect the Helenos GCC patch (https://github.com/HelenOS/gcc/commit/78c2ab486d3a655b6150b6a85a54282f183cec54)
- What is `.gnu.hash`? https://flapenguin.me/elf-dt-hash , https://flapenguin.me/elf-dt-gnu-hash
- https://sunshowers.io/posts/rustc-segfault-illumos/
- https://users.rust-lang.org/t/clarifications-on-rusts-relationship-to-libc/56767

- https://stackoverflow.com/questions/14530590/implementing-thread-local-storage-in-custom-libc

Exception handling (and what the hell is eh_personality):

- https://doc.rust-lang.org/1.2.0/src/std/rt/unwind/mod.rs.html#11-319
- https://llvm.org/docs/ExceptionHandling.html
- https://nicolasbrailo.github.io/blog/history.html articles from early 2013
- https://wiki.osdev.org/Libsupcxx
- https://maskray.me/blog/2020-12-12-c++-exception-handling-abi (level 1/2)

TLS:

- https://maskray.me/blog/2021-02-14-all-about-thread-local-storage
- https://users.cs.jmu.edu/bernstdh/web/common/lectures/summary_unix_pthreads_thread-specific-data.php
- https://www.akkadia.org/drepper/tls.pdf
- https://refspecs.linuxbase.org/elf/gabi4+/ch5.pheader.html
