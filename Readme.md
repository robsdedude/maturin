# Pyo3-pack

[![Linux and Mac Build Status](https://img.shields.io/travis/PyO3/pyo3-pack/master.svg?style=flat-square)](https://travis-ci.org/PyO3/pyo3-pack)
[![Windows Build status](https://img.shields.io/appveyor/ci/konstin/pyo3-pack/master.svg?style=flat-square)](https://ci.appveyor.com/project/konstin/pyo3-pack/branch/master)
[![Crates.io](https://img.shields.io/crates/v/pyo3-pack.svg?style=flat-square)](https://crates.io/crates/pyo3-pack)
[![PyPI](https://img.shields.io/pypi/v/pyo3-pack.svg?style=flat-square)](https://pypi.org/project/pyo3-pack/)
[![Chat on Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?style=flat-square)](https://gitter.im/PyO3/Lobby)

Build and publish crates with pyo3, rust-cpython and cffi bindings as well as rust binaries as python packages.

This project was meant as a zero configuration replacement for [setuptools-rust](https://github.com/PyO3/setuptools-rust). It supports building wheels for python 2.7 and 3.5+ on windows, linux and mac and can upload them to [pypi](https://pypi.org/).

## Usage

You can either download binaries from the [latest release](https://github.com/PyO3/pyo3-pack/releases/latest) or install it with pip:

```shell
pip install pyo3-pack
```

You can also install pyo3-pack from source, though it's an older version:

```shell
cargo install pyo3-pack
```

There are three main subcommands:

 * `publish` builds the crate into python packages and publishes them to pypi.
 * `build` builds the wheels and stores them in a folder (`target/wheels` by default), but doesn't upload them.
 * `develop` builds the crate and install it's as a python module directly in the current virtualenv

`pyo3` and `rust-cpython` bindings are automatically detected, for cffi or binaries you need to pass `-b cffi` or `-b bin`. pyo3-pack needs no extra configuration files, and also doesn't clash with an existing setuptools-rust or milksnake configuration. You can even integrate it with testing tools such as tox (see `get-fourtytwo` for an example).

The name of the package will be the name of the cargo project, i.e. the name field in the `[package]` section of Cargo.toml. The name of the module, which you are using when importing, will be the `name` value in the `[lib]` section (which defaults to the name of the package). For binaries it's simply the name of the binary generated by cargo.

Pip allows adding so called console scripts, which are shell commands that execute some function in you program. You can add console scripts in a section `[package.metadata.pyo3-pack.scripts]`. The keys are the script names while the values are the path to the function in the format `some.module.path:class.function`, where the `class` part is optional. The function is called with no arguments. Example:

```toml
[package.metadata.pyo3-pack.scripts]
get_42 = "get_fourtytwo:DummyClass.get_42"
```

You can also specify [trove classifiers](https://pypi.org/classifiers/) in your Cargo.toml under `package.metadata.pyo3-pack.classifier`, e.g.:

```toml
[package.metadata.pyo3-pack]
classifier = ["Programming Language :: Python"]
```

## pyo3 and rust-cpython

For pyo3 and rust-cpython, pyo3-pack can only build packages for installed python versions, so you might want to use pyenv, deadsnakes or docker for building. If you don't set your own interpreters with `-i`, a heuristic is used to search for python installations. You can get a list all found versions with the `list-python` subcommand.


## Cffi

 Cffi wheels are compatible with all python versions, but they need to have `cffi` installed for the python used for building (`pip install cffi`).

 Until [eqrion/cbdingen#203](https://github.com/eqrion/cbindgen/issues/203) is resolved, you also need cbindgen 0.6.4 as build dependency and a build script that writes c headers to a file called `target/header.h`:

```rust
extern crate cbindgen;

use std::env;
use std::path::Path;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let bindings = cbindgen::Builder::new()
        .with_no_includes()
        .with_language(cbindgen::Language::C)
        .with_crate(crate_dir)
        .generate()
        .unwrap();
    bindings.write_to_file(Path::new("target").join("header.h"));
}
```

## Manylinux and auditwheel

For portability reasons, native python modules on linux must only dynamically link a set of very few libraries which are installed basically everywhere, hence the name manylinux. The pypa offers a special docker container and a tool called [auditwheel](https://github.com/pypa/auditwheel/) to ensure compliance with the [manylinux rules](https://www.python.org/dev/peps/pep-0513/#the-manylinux1-policy). pyo3-pack contains a reimplementation of the most important part of auditwheel that checks the generated library, so there's no need to use external tools. If you want to disable the manylinux compliance checks for some reason, use the `--skip-auditwheel` flag.

pyo3-pack itself is manylinux compliant when compiled with the `musl` feature and a musl target, which is true for the version published on pypi. The binaries on the release pages have keyring integration (though the `password-storage` feature), which is not manylinux compliant.

### Build

```
USAGE:
    pyo3-pack build [FLAGS] [OPTIONS]

FLAGS:
    -h, --help               Prints help information
        --release            Pass --release to cargo
        --skip-auditwheel    Don't check for manylinux compliance
        --strip              Strip the library for minimum file size
    -V, --version            Prints version information

OPTIONS:
    -m, --manifest-path <PATH>                      The path to the Cargo.toml [default: Cargo.toml]
        --target <TRIPLE>                           The --target option for cargo
    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`
```

### Publish

```
USAGE:
    pyo3-pack publish [FLAGS] [OPTIONS]

FLAGS:
        --debug              Do not pass --release to cargo
    -h, --help               Prints help information
        --no-strip           Strip the library for minimum file size
        --skip-auditwheel    Don't check for manylinux compliance
    -V, --version            Prints version information

OPTIONS:
    -m, --manifest-path <PATH>                      The path to the Cargo.toml [default: Cargo.toml]
        --target <TRIPLE>                           The --target option for cargo
    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
    -p, --password <password>
            Password for pypi or your custom registry. Note that you can also pass the password through
            PYO3_PACK_PASSWORD
    -r, --repository-url <registry>
            The url of registry where the wheels are uploaded to [default: https://upload.pypi.org/legacy/]

        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

    -u, --username <username>                       Username for pypi or your custom registry
```

### Develop

```
USAGE:
    pyo3-pack develop [FLAGS] [OPTIONS]

FLAGS:
    -h, --help       Prints help information
        --release    Pass --release to cargo
        --strip      Strip the library for minimum file size
    -V, --version    Prints version information

OPTIONS:
    -b, --bindings <binding_crate>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

    -m, --manifest-path <manifest_path>             The path to the Cargo.toml [default: Cargo.toml]
        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`
```

## Code

The main part is the pyo3-pack library, which is completely documented and should be well integratable. The accompanying `main.rs` takes care username and password for the pypi upload and otherwise calls into the library.

There are three different examples, which are also used for integration testing: `get_fourtytwo` with pyo3 bindings, `points` crate with cffi bindings and `hello-world` as a binary. The `sysconfig` folder contains the output of `python -m sysconfig` for different python versions and platform, which is helpful during development.

You need to install `virtualenv` and `cffi` (`pip install virtualenv cffi`) to run the tests.

You might want to have look into my [blog post](https://blog.schuetze.link/2018/07/21/a-dive-into-packaging-native-python-extensions.html) which explains the intricacies of building native python packages.
