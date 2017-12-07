[base64](https://crates.io/crates/base64)
===

It's base64. What more could anyone want?

Example
---

In Cargo.toml: `base64 = "~0.6.0"`

```rust
extern crate base64;

use base64::{encode, decode};

fn main() {
    let a = b"hello world";
    let b = "aGVsbG8gd29ybGQ=";

    assert_eq!(encode(a), b);
    assert_eq!(a, &decode(b).unwrap()[..]);
}
```

API
---

base64 exposes six functions:

```rust
fn encode<T: ?Sized + AsRef<[u8]>>(&T) -> String;
fn decode<T: ?Sized + AsRef<[u8]>>(&T) -> Result<Vec<u8>, DecodeError>;
fn encode_config<T: ?Sized + AsRef<[u8]>>(&T, Config) -> String;
fn encode_config_buf<T: ?Sized + AsRef<[u8]>>(&T, Config, &mut String);
fn decode_config<T: ?Sized + AsRef<[u8]>>(&T, Config) -> Result<Vec<u8>, DecodeError>;
fn decode_config_buf<T: ?Sized + AsRef<[u8]>>(&T, Config, &mut Vec<u8>) -> Result<(), DecodeError>;
```

`STANDARD`, `URL_SAFE`, `URL_SAFE_NO_PAD`, and `MIME` configuation structs are provided for convenience. `encode` and `decode` are convenience wrappers for the `_config` functions called with the `STANDARD` config, and they are themselves wrappers of the `_buf` functions that allocate on the user's behalf. Encode produces valid padding absent a config that states otherwise; decode produces the same output for valid or omitted padding in all cases, but errors on invalid (superfluous) padding. Whitespace in the input to decode is an error for all modes except `MIME`, which disregards it ("whitespace" according to POSIX-locale `isspace`, meaning \n \r \f \t \v and space).

`Config` exposes a constructor to allow custom combinations of character set, output padding, input whitespace permissiveness, linewrapping, and line ending character(s). The vast majority of usecases should be covered by the four provided, however.

Purpose
---

I have a fondness for small dependency footprints, ecosystems where you can pick and choose what functionality you need, and no more. Unix philosophy sort of thing I guess, many tiny utilities interoperating across a common interface. One time making a Twitter bot, I ran into the need to correctly pluralize arbitrary words. I found on npm a module that did nothing but pluralize words. Nothing else, just a couple of functions. I'd like for this to be that "just a couple of functions."

Developing
---

Benchmarks are in `benches/`. Running them requires nightly rust, but `rustup` makes it easy:

```
rustup run nightly cargo bench
```

Decoding is aided by some pre-calculated tables, which are generated by:

```
cargo run --example make_tables > src/tables.rs.tmp && mv src/tables.rs.tmp src/tables.rs
```

Profiling
---

On Linux, you can use [perf](https://perf.wiki.kernel.org/index.php/Main_Page) for profiling. First, enable debug symbols in Cargo.toml. Don't commit this change, though, since it's usually not what you want (and costs some performance):

```
[profile.release]
debug = true
```

Then compile the benchmarks. (Just re-run them and ^C once the benchmarks start running; all that's needed is to recompile them.)

Run the benchmark binary with `perf` (shown here filtering to one particular benchmark, which will make the results easier to read). `perf` is only available to the root user on most systems as it fiddles with event counters in your CPU, so use `sudo`. We need to run the actual benchmark binary, hence the path into `target`. You can see the actual full path with `rustup run nightly cargo bench -v`; it will print out the commands it runs. If you use the exact path that `bench` outputs, make sure you get the one that's for the benchmarks, not the tests. You may also want to `cargo clean` so you have only one `benchmarks-` binary (they tend to accumulate).

```
sudo perf record target/release/deps/benchmarks-* --bench decode_10mib_reuse
```

Then analyze the results, again with perf:

```
sudo perf annotate -l
```

You'll see a bunch of interleaved rust source and assembly like this. The section with `lib.rs:327` is telling us that 4.02% of samples saw the `movzbl` aka bit shift as the active instruction. However, this percentage is not as exact as it seems due to a phenomenon called *skid*. Basically, a consequence of how fancy modern CPUs are is that this sort of instruction profiling is inherently inaccurate, especially in branch-heavy code.

```
 lib.rs:322    0.70 :     10698:       mov    %rdi,%rax
    2.82 :        1069b:       shr    $0x38,%rax
         :                  if morsel == decode_tables::INVALID_VALUE {
         :                      bad_byte_index = input_index;
         :                      break;
         :                  };
         :                  accum = (morsel as u64) << 58;
 lib.rs:327    4.02 :     1069f:       movzbl (%r9,%rax,1),%r15d
         :              // fast loop of 8 bytes at a time
         :              while input_index < length_of_full_chunks {
         :                  let mut accum: u64;
         :
         :                  let input_chunk = BigEndian::read_u64(&input_bytes[input_index..(input_index + 8)]);
         :                  morsel = decode_table[(input_chunk >> 56) as usize];
 lib.rs:322    3.68 :     106a4:       cmp    $0xff,%r15
         :                  if morsel == decode_tables::INVALID_VALUE {
    0.00 :        106ab:       je     1090e <base64::decode_config_buf::hbf68a45fefa299c1+0x46e>
```


Fuzzing
---

This uses [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz). See `fuzz/fuzzers` for the available fuzzing scripts. To run, use an invocation like these:

```
rustup run nightly cargo fuzz run roundtrip
rustup run nightly cargo fuzz run roundtrip_no_pad
rustup run nightly cargo fuzz run roundtrip_mime -- -max_len=10240
rustup run nightly cargo fuzz run roundtrip_random_config -- -max_len=10240
```


License
---

This project is dual-licensed under MIT and Apache 2.0.