"""
cargo-raze crate build file.

DO NOT EDIT! Replaced on runs of cargo-raze
"""
package(default_visibility = [
  # Public for visibility by "@raze__crate__version//" targets.
  #
  # Prefer access through "//third_party/cargo", which limits external
  # visibility to explicit Cargo.toml dependencies.
  "//visibility:public",
])

licenses([
  "notice", # "MIT"
  "unencumbered", # "Unlicense"
])

load(
    "@io_bazel_rules_rust//rust:rust.bzl",
    "rust_library",
    "rust_binary",
    "rust_test",
)


rust_binary(
    # Prefix bin name to disambiguate from (probable) collision with lib name
    # N.B.: The exact form of this is subject to change.
    name = "cargo_bin_aho_corasick_dot",
    crate_root = "src/main.rs",
    srcs = glob(["**/*.rs"]),
    deps = [
        # Binaries get an implicit dependency on their lib
        ":aho_corasick",
        "@raze__memchr__2_0_2//:memchr",
    ],
    rustc_flags = [
        "--cap-lints allow",
        "--target=x86_64-unknown-linux-gnu",
    ],
    version = "0.6.8",
    crate_features = [
    ],
)


rust_library(
    name = "aho_corasick",
    crate_root = "src/lib.rs",
    crate_type = "lib",
    srcs = glob(["**/*.rs"]),
    deps = [
        "@raze__memchr__2_0_2//:memchr",
    ],
    rustc_flags = [
        "--cap-lints allow",
        "--target=x86_64-unknown-linux-gnu",
    ],
    version = "0.6.8",
    crate_features = [
    ],
)

# Unsupported target "bench" with type "bench" omitted
# Unsupported target "dict-search" with type "example" omitted
