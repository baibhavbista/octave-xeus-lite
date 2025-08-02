# Octave Xeus Lite

This repository provides a demonstration of a JupyterLite instance running an Octave kernel, powered by Xeus. This project is templated from the [jupyterlite/xeus-lite-demo](https://github.com/jupyterlite/xeus-lite-demo).

## Current Status

The project is approximately 90% complete. I think all of the necessary builds are ready, and now one just needs to figure out issues like why xeus-lite or jupyterlite is failing silently, or sometimes fetching only the js file and not the wasm file.

I've exceeded my [appetite](https://basecamp.com/shapeup/1.2-chapter-03) for this project for now, so I am going to leave it here for now.

One potential major change that might be required (might be causing the issues) is that our build uses static instead of shared libraries, while all of the other xeus-X-wasm projects seem to use shared libraries. I think static makes more sense for Octave, but shared might work too, idk


## Introduction

### What is Octave?

GNU Octave is a high-level language, primarily intended for numerical computations. It provides a convenient command-line interface for solving linear and nonlinear problems numerically, and for performing other numerical experiments. It is largely compatible with MATLAB. Octave is used in various fields like engineering, signal processing, machine learning, and scientific research.

### What are Jupyter Notebooks?

The Jupyter Notebook is an open-source web application that allows you to create and share documents that contain live code, equations, visualizations, and narrative text. While Jupyter is widely known for its use with Python, it supports many other programming languages, including Julia, R, and, with the right kernel, Octave.

### What is JupyterLite?

JupyterLite is a Jupyter distribution that runs entirely in the web browser, without any server-side components. It is a sub-project of Project Jupyter that aims to provide a lightweight, client-side, and serverless Jupyter experience. Both the Jupyter front-end and the language kernels run in WebAssembly within the browser.

### What is Xeus?

Xeus is a C++ library for writing Jupyter kernels. It provides a flexible and powerful framework for creating kernels for various languages. `xeus-octave` is an existing project that provides a Jupyter kernel for a locally installed Octave build.

## How This Project Works

This project brings Octave to the browser via JupyterLite. The process involves a few key steps:

1.  **Octave for WebAssembly**: We start with a WebAssembly build of Octave. This is achieved by using a modified version of the `octave-wasm` repository, which produces a WebAssembly-enabled binary of the Octave interpreter.
  - https://github.com/baibhavbista/octave-wasm/tree/for-xeus-octave
    - Just run `make build` and then `make extract`
    - `Dockerfile` is the main file

2.  **The Xeus Octave Wasm Kernel**: We then use the `emscripten-forge-recipes` repository to create a new build of `xeus-octave`. This build is specifically designed to be compiled for WebAssembly and to link against the Wasm build of Octave.
  - https://github.com/baibhavbista/emscripten-forge-recipes/tree/octave-bob-3 
    - main folders are `xeus`, `octave-build-static` and `xeus-octave`
      - needed to make changes to `xeus` to force static builds
      - `octave-build-static` just uses the output of the earlier step and converts it into proper form (Important: you have to paste the build output of octave-wasm build to `sources/octave-build-static-remove-fwasm-exceptions.tar.bz2`)
      - `xeus-octave` has some patches on top of the actual [xeus-octave](https://github.com/jupyter-xeus/xeus-octave) (non-WASM) codebase
  ```
  /bin/bash -c "$(curl -fsSL https://pixi.sh/install.sh)" && source ~/.bashrc && pixi run setup

  pixi run build-emscripten-wasm32-pkg recipes/recipes_emscripten/xeus 2>&1 | tee xeus-logfile.txt
  
  pixi run build-emscripten-wasm32-pkg recipes/recipes_emscripten/octave-build-static 2>&1 | tee octave-build-static-logfile.txt

  pixi run build-emscripten-wasm32-pkg recipes/recipes_emscripten/xeus-octave 2>&1 | tee xeus-octave-logfile.txt
  ```

3.  **JupyterLite Integration**: Finally, this repository integrates the WebAssembly-based Octave kernel into a JupyterLite instance, making it possible to run Octave code directly in the browser.


## Key Artifacts

The core of this project relies on two main build artifacts, which are conda packages compiled for the `emscripten-wasm32` platform. They are in a `local-channel` inside the `channels/` directory.

### Artifact 1: `octave-build-static-0.1.0-hc286ada_1.tar.bz2`

This package provides the core **GNU Octave numerical computing environment**, specifically compiled for WebAssembly.

*   **Purpose:** To serve as a self-contained, "headless" version of the Octave interpreter and its extensive mathematical libraries. It is a **static build**, meaning all its components are compiled into archive files (`.a` files), which is essential for being linked into a single, final WebAssembly module. It is the fundamental dependency that provides all the Octave functionality.

*   **Contents:**
    *   The main Octave static libraries (`liboctave.a`, `liboctinterp.a`).
    *   A complete set of pre-compiled static libraries for all of Octave's dependencies, including mathematical libraries (CLAPACK, BLAS, SuiteSparse) and utility libraries (PCRE).
    *   The C/C++ header files required for other programs to compile against the Octave libraries.
    *   The full standard library of Octave `.m` script files.

*   **Production Process:** This artifact was created using a multi-stage `Dockerfile` build process:
    1.  **Toolchain Setup:** A specific version of the Emscripten SDK (`3.1.73`) was installed to ensure a consistent and reproducible C++/C/Fortran to WebAssembly compiler toolchain.
    2.  **Dependency Compilation:** Numerous third-party libraries, including complex Fortran-based code like LAPACK, were compiled from source into static WebAssembly libraries.
    3.  **Octave Compilation:** The GNU Octave source code was configured and compiled with a highly customized build script. This script was tailored to:
        *   Target the `emscripten-wasm32` platform.
        *   Disable features unsuitable for a browser environment, such as threading, dynamic linking, and GUI components.
        *   Use Emscripten's default JavaScript-based exception handling to ensure compatibility with other packages in the ecosystem.
    4.  **Packaging:** All the resulting static libraries, headers, and script files were collected and bundled into the `.tar.bz2` Conda package.

### Artifact 2: `xeus-octave-wasm-0.2.0-h53eb3ed_4.tar.bz2`

This package is the **Jupyter kernel for Octave**, which acts as the bridge between the Jupyter frontend and the Octave interpreter.

*   **Purpose:** To implement the Jupyter wire protocol, allowing a user in a Jupyter environment (like JupyterLite) to send code to the Octave interpreter, execute it, and see the results (text, plots, etc.) displayed in the browser. The `-wasm` suffix signifies that this is the specific version of the kernel compiled to run entirely in the browser.

*   **Contents:**
    *   The primary kernel executables: `xoctave.wasm` (the C++ kernel compiled to WebAssembly) and `xoctave.js` (the JavaScript loader).
    *   A `kernel.json` metadata file, which is critical for Jupyter to discover the kernel and know how to launch it.
    *   **Crucially, this package does *not* contain Octave itself.** It is a small "glue" package that depends on `octave-build-static` to be present at runtime.

*   **Production Process:** This artifact was created using the `emscripten-forge-recipes` build system (`rattler-build`/`pixi`).
    1.  **Environment Setup:** The build system created a temporary, controlled environment containing the Emscripten compiler and all necessary host dependencies, including `xeus`, `xeus-lite`, and the `octave-build-static` package described above.
    2.  **Source Patching:** A large patch file was applied to the `xeus-octave` source code to add WebAssembly-specific logic and workarounds.
    3.  **Compilation and Linking:** The patched C++ code was compiled and then statically linked against the libraries from `octave-build-static`, `xeus`, and `xeus-lite`. This final link step combined everything into the single `xoctave.wasm` file.
    4.  **Metadata Installation:** A custom step in the build script (`build.sh`) manually installed the `kernel.json` file into the correct directory (`share/jupyter/kernels/xoctave/`) within the package structure.
    5.  **Packaging:** The final `.wasm`, `.js`, and `kernel.json` files were bundled into the `.tar.bz2` Conda package.

## Kernel Configuration

The Jupyter kernel is defined by the `kernel.json` file located inside the `xeus-octave-wasm` package at `share/jupyter/kernels/xoctave/kernel.json`. Its content is:

```json
{
    "argv": ["$PREFIX/bin/xoctave", "-f", "{connection_file}"],
    "display_name": "Octave",
    "language": "octave"
}
```

