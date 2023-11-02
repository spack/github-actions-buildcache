# Spack buildcache for Github Actions

This repo provides a buildcache to speed up Spack in your Github Actions.

Currently it provides binaries for `%gcc@12 os=ubuntu22.04 target=x86_64_v2` from Spack `develop`.

To use it, add an environment `spack.yaml` to the root of your own repository

```yaml
spack:
  view: my_view
  specs:
  - python@3.11

  packages:
    all:
      require: '%gcc@12 target=x86_64_v2'

  mirrors:
    spack-buildcache: oci://ghcr.io/haampie/spack-buildcache
```

and Spack install it in a Github Action:

```yaml
name: Build

on: push

env:
  SPACK_COLOR: always
  SPACK_BACKTRACE: please

jobs:
  example:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install Spack
        run: |
          git clone --depth=1 https://github.com/spack/spack.git
          echo "$PWD/spack/bin/" >> "$GITHUB_PATH"

      - name: Concretize
        run: spack -e . concretize

      - name: Install
        run: spack -e . install --no-check-signature

      - name: Run
        run: ./my_view/bin/python -c 'print("hello world")'
```

## Caching your own binaries

If you want to cache your own binaries too, there are three steps to take:

1. Add an additional mirror to your environment config:

   ```yaml
   spack:
     ...
     mirrors:
       spack-buildcache: oci://ghcr.io/haampie/spack-buildcache
       local-buildcache: oci://ghcr.io/<username>/spack-buildcache
   ```

2. Configure the concurrency of the job, as well as permissions for
   `GITHUB_TOKEN`:

   ```yaml
   jobs:
     example:
       runs-on: ubuntu-22.04
       permissions:
         packages: write
       concurrency: update_buildcache
       steps: ...
   ```
   NOTE: Although builds can happen concurrently across different Github
   Actions, it's best to update the index sequentially.

3. Add an extra job step that pushes installed Spack packages to the local
   buildcache:

   ```yaml
   jobs:
     example:
       ...
       steps:
         ...
         - name: Push packages and update index
           run: |
             spack -e . mirror set --push --oci-username <username> --oci-password "${{ secrets.GITHUB_TOKEN }}" local-buildcache
             spack -e . buildcache push --base-image ubuntu:22.04 --unsigned --update-index local-buildcache
           if: always()
   ```
   NOTE: Make sure to add `if: always()`, so that binaries for successfully
   installed packages are available also when a dependent fails to build.


