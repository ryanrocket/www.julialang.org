---
layout: post
title: Pkg + BinaryBuilder -- The Next Generation
author: Elliot Saba, Stefan Karpinski, Kristoffer Carlsson
---

# Pkg + BinaryBuilder -- The Next Generation

Over the past few months, we have been iterating on and refining a design for `Pkg` to natively be able to reason about binary objects that are not Julia packages.  While the motivating application for this work has been improving the installation experience for binaries built with [`BinaryBuilder.jl`](https://github.com/JuliaPackaging/BinaryBuilder.jl), the artifacts subsystem is much more general.

# Pkg Artifacts

Pkg artifacts, as outlined in [`Pkg.jl#1234`](https://github.com/JuliaLang/Pkg.jl/issues/1234), provides a convenient way to associate containers of data with Julia projects and packages.  Artifacts are referred to by content-hash, or optionally by a name that is bound to a hash through an `Artifacts.toml` file.  An example `Artifacts.toml` file is shown here as an example:

```TOML
[socrates]
git-tree-sha1 = "43563e7631a7eafae1f9f8d9d332e3de44ad7239"
lazy = true

    [[socrates.download]]
    url = "https://github.com/staticfloat/small_bin/raw/master/socrates.tar.gz"
    sha256 = "e65d2f13f2085f2c279830e863292312a72930fee5ba3c792b14c33ce5c5cc58"

    [[socrates.download]]
    url = "https://github.com/staticfloat/small_bin/raw/master/socrates.tar.bz2"
    sha256 = "13fc17b97be41763b02cbb80e9d048302cec3bd3d446c2ed6e8210bddcd3ac76"

[[c_simple]]
arch = "x86_64"
git-tree-sha1 = "4bdf4556050cb55b67b211d4e78009aaec378cbc"
libc = "musl"
os = "linux"

    [[c_simple.download]]
    sha256 = "411d6befd49942826ea1e59041bddf7dbb72fb871bb03165bf4e164b13ab5130"
    url = "https://github.com/JuliaBinaryWrappers/c_simple_jll.jl/releases/download/c_simple+v1.2.3+0/c_simple.v1.2.3.x86_64-linux-musl.tar.gz"

[[c_simple]]
arch = "x86_64"
git-tree-sha1 = "51264dbc770cd38aeb15f93536c29dc38c727e4c"
os = "macos"

    [[c_simple.download]]
    sha256 = "6c17d9e1dc95ba86ec7462637824afe7a25b8509cc51453f0eb86eda03ed4dc3"
    url = "https://github.com/JuliaBinaryWrappers/c_simple_jll.jl/releases/download/c_simple+v1.2.3+0/c_simple.v1.2.3.x86_64-apple-darwin14.tar.gz"

[processed_output]
git-tree-sha1 = "1c223e66f1a8e0fae1f9fcb9d3f2e3ce48a82200"
```

This `Artifacts.toml` binds three artifacts; one named `socrates`, one named `c_simple` and one named `processed_output`.  The single required piece of information for an artifact is its `git-tree-sha1`.  Because artifacts are addressed only by their content hash, the purpose of an `Artifacts.toml` file is to provide metadata about these artifacts, such as binding a human-readable name to an appropriate content hash, providing information about where an artifact may be downloaded from, or even binding a single name to multiple hashes, keyed by platform-specific constraints such as operating system or libgfortran version.

## Artifact types and properties

In the above example, the `socrates` artifact showcases a platform-independent artifact with multiple download locations.  When downloading and installing the `socrates` artifact, URLs will be attempted in-order until one succeeds.  The `socrates` artifact is marked as `lazy`, which means that it will not be automatically downloaded when the containing package is installed, but rather will be downloaded on-demand when the package first attempts to use it.

The `c_simple` artifact showcases a platform-dependent artifact, where each entry in the `c_simple` array contains keys that help the calling package choose the appropriate download based on the particulars of the host machine.  Note that each artifact contains both a `git-tree-sha1` and a `sha256` for each download entry.  This is to ensure that the downloaded tarball is secure before attempting to unpack it, as well as enforcing that all tarballs must expand to the same overall tree hash.

The `processed_output` artifact contains no `download` stanza, and so cannot be installed.  An artifact such as this would be the result of code that was previously run, generating a new artifact and binding the resultant hash to a name within this project.

## Working with artifacts

Artifacts can be manipulated using convenient APIs exposed from the `Pkg.Artifacts` namespace.  As a motivating example, let us imagine that we are writing a package that needs to load the [Iris machine learning dataset](https://archive.ics.uci.edu/ml/datasets/iris). While we could just download the dataset during a build step into the package directory, and many packages currently do precisely this, that has some significant drawbacks. First, it modifies the package directory, making package installation stateful, which we'd like to avoid. In the future, we would like to reach the point where packages can be installed completely read-only, instead of being able to modify themselves after installation. Second, the downloaded data is not shared across different versions of our package. If we have three different versions of the package installed for use by various projects, then we need three different copies of the data, even if it hasn't changed between those versions. Moreover, each time we upgrade or downgrade the package, unless we do something clever (and probably brittle), we will have to download the data again.  With artifacts, we will instead check to see if our `iris` artifact already exists on-disk and only if it doesn't will we download and install it, after which we can bind the result into our `Artifacts.toml` file:

```julia
using Pkg.Artifacts

# This is the path to the Artifacts.toml we will manipulate
artifact_toml = joinpath(@__DIR__, "Artifacts.toml")

# Query the `Artifacts.toml` file for the hash bound to the name "iris"
# (returns `nothing` if no such binding exists)
iris_hash = artifact_hash("iris", artifact_toml)

# If the name was not bound, or the hash it was bound to does not exist, create it!
if iris_hash == nothing || !artifact_exists(iris_hash)
    # create_artifact() returns the content-hash of the artifact directory once we're finished creating it
    iris_hash = create_artifact() do artifact_dir
        # We create the artifact by simply downloading a few files into the new artifact directory
        iris_url_base = "https://archive.ics.uci.edu/ml/machine-learning-databases/iris"
        download("$(iris_url_base)/iris.data", joinpath(artifact_dir, "iris.csv"))
        download("$(iris_url_base)/bezdekIris.data", joinpath(artifact_dir, "bezdekIris.csv"))
        download("$(iris_url_base)/iris.names", joinpath(artifact_dir, "iris.names"))
    end

    # Now bind that hash within our `Artifacts.toml`.  `force = true` means that if it already exists,
    # just overwrite with the new content-hash.  Unless the source files change, we do not expect
    # the content hash to change, so this should not cause unnecessary version control churn.
    bind_artifact("iris", iris_hash, artifact_toml)
end

# Get the path of the iris dataset, either newly created or previously generated.
# this should be something like `~/.julia/artifacts/dbd04e28be047a54fbe9bf67e934be5b5e0d357a`
iris_dataset_path = artifact_path(iris_hash)
```

For the specific use case of using artifacts that were previously bound, we have the shorthand notation `artifact"name"` which will automatically search for the `Artifacts.toml` file contained within the current package, look up the given artifact by name, install it if it is not yet installed, then return the path to that given artifact.

# BinaryBuilder.jl

You may be able to guess that `BinaryBuilder.jl` knows how to generate `Artifacts.toml` files ([example](https://github.com/JuliaBinaryWrappers/c_simple_jll.jl/blob/master/Artifacts.toml)), but that's not the only change.  We were tired of the current best practice of needing to manually express the graph of all dependencies within your package's `deps/build.jl` file.  To date, there has not been a simple, concise method of recursively installing your binary dependencies, users have been forced to resort to strategies such as embedding the `build.jl` files of all binary dependencies within their own packages.  This works, but it's way clunkier than we'd like.  Luckily, we already have a package manager that knows how to deal with recursive dependencies, and so a simple solution was devised; as a part of the output of a `BinaryBuilder.jl` run, we will generate a wrapper julia package that will simultaneously allow us to express the DAG of binary dependencies, as well as provide boilerplate julia wrapper code that will make dealing with libraries and executables much simpler.  We refer to these autogenerated packages as "jll" packages.

## Julia library (jll) packages

Autogenerated BinaryBulider-produced packages are normal Julia packages, with the notable inclusion of `Artifacts.toml` files that download the appropriate version of whatever binary artifacts were built by `BinaryBuilder` and uploaded to the repository's GitHub release. We refer to these autogenerated packages as "Julia library packages" or "jll"s for short.  BinaryBuilder attempts to upload all packages to `JuliaBinaryWrappers/$(package_name)_jll.jl`, however this is, of course, configurable.  An example package [is given here](https://github.com/JuliaBinaryWrappers/c_simple_jll.jl), the most interesting part being the new API that is exposed by these autogenerated packages.

The code bindings within jll packages are autogenerated from the `Products` defined within the `build_tarballs.jl` file that generates the package.  For example purposes, we will assume that the following products were defined:

```julia
products = [
    FileProduct("src/data.txt", :data_txt),
    LibraryProduct("libdataproc", :libdataproc),
    ExecutableProduct("mungify", :mungify_exe)
]
```

With such products defined, the jll package will contain `data_txt`, `libdataproc` and `mungify_exe` symbols exported. For `FileProduct` variables, the exported value is a string pointing to the location of the file on-disk.  For `LibraryProduct` variables, it is a string corresponding to the `SONAME` of the desired library (it will have already been `dlopen()`'ed during the `__init__()` method of the jll package module so typical `ccall()` usage applies), and for `ExecutableProduct` variables, the exported value is a function that can be called to set appropriate environment variables such as `PATH` and `LD_LIBRARY_PATH`.  This is necessary so that nested depdenencies works properly, e.g. `ffmpeg` calling the `x264` binary during video encoding.  Example:

```julia
using c_simple_jll

# For file products, you can access its file location directly:
data_lines = open(data_txt, "r") do io
    readlines(io)
end

# For library products, you can use the exported variable name in `ccall()` invocations directly
num_chars = ccall((libdataproc, :count_characters), Cint, (Cstring, Cint), data_lines[1], length(data_lines[1]))

# For executable products, you can use the exported variable name as a function that you can call
mungify_exe() do mungify_exe_path
    run(`$mungify_exe_path $num_chars`)
end
```

One of the big benefits of this new system is that running binaries that must link against libraries provided by another dependency, or call binaries provided within another dependency all works flawlessly because the wrapper jll packages automatically set up the appropriate environment variables.

## Changes to `build_tarballs.jl`

Not much has changed here, the first is some cleanup of the clunky products API; previously you needed to provide a closure to curry a `prefix` through to your products; this is no longer the case.  To make this explicit, while you would previously write declarations such as:

```julia
products(prefix) = [
    LibraryProduct(prefix, "libglib", :libglib)
]
```

You now simply write:
```julia
products = [
    LibraryProduct("libglib", :libglib)
]
```

The second big change is that dependencies are no longer linked straight to `build.jl` files, rather you give the names of the jll packages you want to install, and they (along with all recursive dependencies) will be installed and symlinked into `${prefix}`, similarly to before.  To make this explicit, while you would previously write declarations such as:

```julia
dependencies = [
    "https://github.com/JuliaBinaryWrappers/c_simple_jll.jl/releases/download/c_simple%2Bv1.2.3%2B7/build_c_simple.v1.2.3.jl",
]
```

You now simply write:
```julia
dependencies = [
    "c_simple_jll",
]
```

Because jll packages are registered just like any other public package, installing these build dependencies is as simple as querying the registry, cloning the latest version of the jll package, inspecting its `Artifacts.toml` file and unpacking the appropriate artifact within the build prefix.  Even better, these build dependencies are automatically recorded as dependencies for the generated new jll package.

# Updating your packages

To update your packages, first build with the latest version of `BinaryBuilder`, then add the autogenerated `jll` packages as dependencies in your projects and packages, use the new API for your `ccall()`'s and `run()`'ing your binaries, and give a small smile as you delete your `deps/build.jl` file.  Death to all global mutable state!

# TODO

* Need more tests
* Need to implement `pkg> gc` for artifacts
