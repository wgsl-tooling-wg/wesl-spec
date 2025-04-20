# Versioning Enhanced WGSL

As we stabilize for 1.0, we should mark a way for tools (and users) to distinguish our first stable version
from subsequent releases.

Issues:

* Feature flags or versions? Consider also version
* perhaps `version WESL2024`?
* We'd like tools to maintain backward compatibility with previous stable versions. What do tools need?
* Look to Rust [editions](https://doc.rust-lang.org/edition-guide/editions/) as a model for versioning
* Look at WGSL discussion of similar topics, e.g. #2941
