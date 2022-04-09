# `opam-esbuild`

[esbuild](https://esbuild.github.io/) is a JavaScript and CSS bundler and minifier.

This opam package wraps [the standalone executable version](https://esbuild.github.io/getting-started/#download-a-build) of the esbuild project. These executables are platform specific. Upon installation, opam will automatically pick the correct executable for your platform and install it as a `esbuild` binary. Supported platforms are Linux arm64, Linux x64, macOS arm64, macOS x64, and Windows x64.

Once installed, you can execute the binary with:

```
opam exec -- esbuild
```

Here is a dune rule that uses the installed binary to bundle a JavaScript file:

```
(rule
 (targets main.js)
 (deps
  (:input main.bc.js))
 (action
  (chdir
   %{workspace_root}
   (run
    esbuild
    --platform=browser
    --external:fs
    --external:tty
    --external:child_process
    --external:constants
    --minify
    --bundle
    --outfile=%{targets}
    %{input}))))
```

## Installation

You can install `opam-esbuild` using `opam`:

```bash
opam install esbuild
```

## License

`opam-esbuild` is released under the [ISC License](https://opensource.org/licenses/ISC).
Esbuild is released under the [MIT License](https://opensource.org/licenses/MIT).
