# 1.4.0 (2026-07-22)


### Features

* Added auth for account signups and account linking (2049602)

# 1.3.0 (2026-07-17)


### Features

* added raid account creation date to account page (c486c0d)

# 1.2.0 (2026-07-16)


### Bug Fixes

* bump built-for game version to 11.67 and sync IL2CPP layout (759e861)


### Features

* great hall improvements to show current bonus, and upgrade cost. (17f11fa)

# 1.1.0 (2026-07-13)


### Features

* added more currency items to the collection (42dffbc)

## 1.0.7 (2026-07-13)


### Bug Fixes

* Changelog should now generate the correct content on release. (b0c1c41)

## 1.0.6 (2026-07-13)


### Bug Fixes

* use git rather than Github for the ChangeLog generation in the release process (9cc6e28)

## 1.0.5 (2026-07-13)


### Bug Fixes

* release building for the goreleaser disabled (ba860d1)
* Raid not running triggered the incorrect version popup (5488e93)

## 1.0.4 (2026-07-12)


### Bug Fixes

* release action (53262f9)

## 1.0.3 (2026-07-10)


### Bug Fixes

* raid-collector-rs/rust-toolchain.toml pins a bare channel = "stable" with no host triple (71bd701)
* added application icon (9cbbb5f)

## 1.0.2 (2026-07-10)


### Bug Fixes

* replaced the dtolnay/rust-toolchain + cross-target approach with a direct rustup toolchain install stable-x86_64-pc-windows-gnu (a real, separate GNU-host toolchain) before switching to it with rustup default. That's what makes cargo build --release actually pick up raid-collector-rs/.cargo/config.toml's MinGW linker settings and produce libraid_collector.dll.a. (f498f88)

## 1.0.1 (2026-07-10)


### Bug Fixes

* release workflow bug fix (289576a)

## 1.0.0 (2026-07-10)


### Features

* added installer for Raid Aegis Daemon (c55c023)
* added avatar and frame to account page (b502e93)
* dungeon teams area added (a35daaa)
* Boss battles data collected - stage 1 (e022f57)
* Added mastery to champion page (2723e60)
* updated Raid client to 11.65.0 (e05a58b)
* Great hall, Inbox, Resources, Chimera, Master information now captured. (5c497f7)
* **layout:** add experience, full_experience, static_gameplay_data offsets (56e623f)
* **item-26:** update layout JSON and header for static hero types (a53f4c9)


### Bug Fixes

* resolve hero names reliably to allow ticks to push data correctly. (742c5be)
* correctly dumped before updating this time (b6978ed)
* ascended, awakened levels should now work as expected (11d4343)
* **push:** treat 3xx as a config error not success (ddd2983)
* **push:** load config from exe dir; block HTTP→HTTPS redirect (a8c5314)
