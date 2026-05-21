# orca-slicer-ffi

A flat C API over OrcaSlicer's `libslic3r`, plus Rust bindings, intended as a
foundation for building alternative slicer user interfaces (initially in Rust)
that drive libslic3r's slicing engine without inheriting OrcaSlicer's
preset/profile system.

OrcaSlicer is included as a vanilla git submodule under `external/OrcaSlicer`.
This repo contains only the FFI shim and bindings; the engine itself lives
upstream.

## Layout

```
ffi/                  C shim (libslic3r_ffi.so)
bindings/rust/        Rust crate + bindgen build script
external/OrcaSlicer/  upstream OrcaSlicer (submodule, pinned)
scripts/build.sh      convenience wrapper
```

## Build

First-time setup, on Linux:

```bash
git submodule update --init --recursive
./scripts/build.sh deps     # ~30 min, one-time — builds OrcaSlicer's deps tree
./scripts/build.sh build    # builds libslic3r + libslic3r_ffi.so
./scripts/build.sh smoke    # runs the Rust introspect + slice smoke tests
```

Or as one shot: `./scripts/build.sh all`.

The shim builds with `SLIC3R_GUI=OFF`. wxWidgets and the OrcaSlicer GUI are not
required.

## API surface

The C side (`ffi/slic3r_ffi.h`) is 15 functions:

- Lifecycle: `slic3r_init`, `slic3r_version`, `slic3r_string_free`
- Option introspection: `slic3r_option_def_count`, `slic3r_option_def_at`,
  `slic3r_option_def_lookup`
- Config: `slic3r_config_new/free/set/get/validate`
- Model: `slic3r_model_new/free/load`
- Slice: `slic3r_slice`

All ~737 libslic3r options are reachable through the string-keyed `config_set`;
the introspection API exposes type, label, tooltip, category, mode, default,
and enum choices for each, suitable for driving a typed UI schema.

The Rust crate (`bindings/rust`) wraps this with safe types. See
`bindings/rust/examples/introspect.rs` for the option-enumeration pattern and
`bindings/rust/examples/slice.rs` for end-to-end slicing.

## Upgrading OrcaSlicer

```bash
cd external/OrcaSlicer
git fetch
git checkout <new-commit-or-tag>
cd ../..
./scripts/build.sh build       # rebuild against the new upstream
./scripts/build.sh smoke       # confirm the shim still works
git add external/OrcaSlicer
git commit -m "Bump OrcaSlicer to <ref>"
```

The submodule pin is the only thing recorded in this repo. There is no fork
of OrcaSlicer to maintain.

## License

AGPL-3.0-or-later, matching upstream OrcaSlicer. Anything that links
`libslic3r_ffi.so` is a derivative work and must also be AGPL-compatible.

## Known limitations

- **`coEnums` defaults are not surfaced.** libslic3r's vector-of-enums
  serializer reads a per-option `keys_map` that's null on the def's cloned
  default value, even though the same mapping is on the def itself.
  9 options affected; `enum_values`/`enum_labels` still work.
- **boost::log goes to stderr by default.** A sink-redirect API is a planned
  addition.
- **Linux only for now.** macOS should work with small CMake tweaks; Windows
  would need symbol-visibility annotations on the C API since MSVC doesn't
  default-export.
