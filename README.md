# Spack buildcache for GitHub Actions

This repo provides a buildcache to speed up Spack in your GitHub Actions.

Currently it provides binaries from Spack `develop` for

- `%gcc@12 os=ubuntu22.04 target=x86_64_v2`
- `%gcc@11 os=ubuntu22.04 target=x86_64_v2`

(gcc@13 binaries are currently not available due to missing gfortran compilers)

To use it, add an environment `spack.yaml` to the root of your own repository

```yaml
spack:
  view: my_view
  specs:
  - python@3.11

  config:
    install_tree:
      root: /opt/spack

  packages:
    all:
      require: '%gcc@12 target=x86_64_v2'

  mirrors:
    spack-buildcache: oci://ghcr.io/spack/github-actions-buildcache
```

and Spack install it in a GitHub Action:

```yaml
name: Build

on: push

env:
  SPACK_COLOR: always

jobs:
  example:
    runs-on: ubuntu-22.04
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Checkout Spack
      uses: actions/checkout@v3
      with:
        repository: spack/spack
        path: spack

    - name: Setup Spack
      run: echo "$PWD/spack/bin" >> "$GITHUB_PATH"

    - name: Concretize
      run: spack -e . concretize

    - name: Install
      run: spack -e . install --no-check-signature

    - name: Run
        run: ./my_view/bin/python -c 'print("hello world")'
```

## Caching your own binaries

If you want to cache your own binaries too, there are three steps to take:

1. Use padding in the install root, and add an additional mirror to `spack.yaml`:

   ```yaml
   spack:
     config:
       install_tree:
         root: /opt/spack
         padded_length: 128
     mirrors:
       spack-buildcache: oci://ghcr.io/spack/github-actions-buildcache
       local-buildcache: oci://ghcr.io/<username>/spack-buildcache
   ```

2. Configure the permissions for `GITHUB_TOKEN`:

   ```yaml
   jobs:
     example:
       runs-on: ubuntu-22.04
       permissions:
         packages: write
   ```

3. Add an extra job step that pushes installed Spack packages to the local
   buildcache:

   ```yaml
   jobs:
     example:
       steps:
       - name: Push packages and update index
         run: |
           spack -e . mirror set --push --oci-username ${{ github.actor }} --oci-password "${{ secrets.GITHUB_TOKEN }}" local-buildcache
           spack -e . buildcache push --base-image ubuntu:22.04 --unsigned --update-index local-buildcache
         if: ${{ !cancelled() }}
   ```
   NOTE: Make sure to add `if: ${{ !cancelled() }}`, so that binaries for successfully
   installed packages are available also when a dependent fails to build.

### Caching your own binaries in *private* repos and buildcaches

When your local buildcache is stored in a private GitHub package,
you need to specify the OCI credentials already *before* `spack concretize`.
This is because Spack needs to fetch the buildcache index. Also, remember to
remove the `--push` flag from `spack mirror set`, since fetching needs
credentials too:

```yaml
jobs:
  example-private:
    steps:
    - name: Login
      run: spack -e . mirror set --oci-username ${{ github.actor }} --oci-password "${{ secrets.GITHUB_TOKEN }}" local-buildcache

    - name: Concretize
      run: spack -e . concretize

    - name: Install
      run: spack -e . install --no-check-signature

    - name: Push packages and update index
      run: spack -e . buildcache push --base-image ubuntu:22.04 --unsigned --update-index local-buildcache
```

From a security perspective, notice that the `GITHUB_TOKEN` is exposed to every
subsequent job step. (This is no different from `docker login`, which also likes
to store credentials in the home directory.)

## Contributing

If you want to make more packages available, contribute to
[spack.yaml](spack.yaml).

## Build strategy

Since compiling software in GitHub actions is relatively slow, this stack is
built using `concretizer:reuse:dependencies`. That means that the latest
versions of the packages listed in [spack.yaml](spack.yaml) are built, but
their dependencies are only updated when a package compatibility rule requires
it. The stack is currently built on demand, not on a schdule.

## License

This project is part of Spack. Spack is distributed under the terms of both the
MIT license and the Apache License (Version 2.0). Users may choose either
license, at their option.

All new contributions must be made under both the MIT and Apache-2.0 licenses.

See LICENSE-MIT, LICENSE-APACHE, COPYRIGHT, and NOTICE for details.

SPDX-License-Identifier: (Apache-2.0 OR MIT)

LLNL-CODE-811652
