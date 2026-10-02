# clang-format

Container images with [clang-format](https://clang.llvm.org/docs/ClangFormat.html) from the LLVM project, the formatter for C, C++, Objective-C, C#, Java, JavaScript, JSON and Protocol Buffers code, together with `git clang-format` and `clang-format-diff.py`, which format only the lines a change touches. clang-format is installed from the [clang-format wheels on PyPI](https://pypi.org/project/clang-format/) into a virtual environment on Ubuntu or Alpine. The images are rebuilt when a new wheel is published and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the LLVM project or the [clang-format wheel project](https://github.com/ssciwr/clang-format-wheel). Report problems with the image in this repository, problems with clang-format itself in the [LLVM issue tracker](https://github.com/llvm/llvm-project/issues), and problems with the wheels [in their repository](https://github.com/ssciwr/clang-format-wheel/issues).

## Quick start

Format the C and C++ files of a git repository in place, from its root:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/clang-format -i $(git ls-files '*.c' '*.cpp' '*.h')
```

clang-format takes the style from the nearest `.clang-format` or `_clang-format` above each file and uses the LLVM style when there is none.

Check the files without changing them, as a CI step. With `--dry-run --Werror`, clang-format prints an `error: code should be clang-formatted [-Wclang-format-violations]` line for each place that would change and exits with a non-zero status:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/clang-format --dry-run --Werror $(git ls-files '*.c' '*.cpp' '*.h')
```

Format code from standard input. `--assume-filename` sets the language and the place to look for `.clang-format`; with nothing mounted there is none, so pass a style:

```sh
docker run --rm -i ghcr.io/randomcontainers/clang-format --style=Google --assume-filename=main.cpp < main.cpp
```

clang-format's output changes between LLVM releases. Pin an exact version tag, such as `ghcr.io/randomcontainers/clang-format:<version>`, or a digest in CI, and format with the same one locally (see [Tags](#tags)).

## Formatting changed lines only

`git clang-format` formats only the lines that differ between the working tree and a commit. With `--diff` it prints the changes instead of making them, and exits with 1 when there are any. To check the lines a branch changed, compare with the commit where it left `origin/main`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint git ghcr.io/randomcontainers/clang-format clang-format --diff "$(git merge-base origin/main HEAD)"
```

Comparing with `origin/main` directly would also report lines that changed on `main` after the branch was created. Without `--diff`, `git clang-format` rewrites the lines in the working tree. The repository must contain the commit you compare against: in GitHub Actions, check out with `fetch-depth: 0` or deep enough to reach it.

`clang-format-diff.py`, also installed as `clang-format-diff`, reads a diff on standard input instead, so git does not have to run in the container. Add `-i` to apply the changes:

```sh
git diff -U0 --no-color "$(git merge-base origin/main HEAD)" | docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint clang-format-diff.py ghcr.io/randomcontainers/clang-format -p1
```

`GIT_CONFIG_SYSTEM` points git at `/usr/local/etc/gitconfig`, which adds `/work` and `/work/*` to `safe.directory`, so git works in a repository mounted at `/work` or below it whatever UID owns it. A repository mounted elsewhere must be owned by the user the container runs as. Pass `-e GIT_CONFIG_NOSYSTEM=1` to turn the setting off.

## Configuration

Mount the directory that holds `.clang-format`, usually the root of your repository, since clang-format looks for it only in the directory of each file and the directories above it. Other ways to choose a style:

- `--style=file:path/to/config`, with the file under `/work`.
- A predefined style such as `--style=Google`, or inline options such as `--style="{BasedOnStyle: LLVM, IndentWidth: 8}"`.
- `--dump-config` prints the complete configuration of a style, as a starting point for your own file: `docker run --rm ghcr.io/randomcontainers/clang-format --style=Google --dump-config > .clang-format`.

A `.clang-format-ignore` file lists files for clang-format to skip. The [style options](https://clang.llvm.org/docs/ClangFormatStyleOptions.html) change with each LLVM release, and clang-format stops with an error on an option it does not know, so a configuration written for a newer clang-format may not work with an older image.

## What is in the image

- `clang-format` in `/usr/local/bin`, a link to the binary in the wheel. It runs without starting Python.
- `git-clang-format`, which git runs for `git clang-format`, and `clang-format-diff.py`, also linked as `clang-format-diff`. Both are Python scripts from the wheel and run from a virtual environment in `/usr/local/lib/clang-format` that uses the distro's `python3`.
- git from the distro, which `git clang-format` needs. On Ubuntu it brings Perl with it. Its system configuration is `/usr/local/etc/gitconfig`, described above.
- pip, which stays in the virtual environment.
- On Alpine, `libstdc++` and `libgcc_s`, which the musl wheel carries in `clang_format.libs` next to the binary. On Ubuntu the binary uses the distro's `libstdc++6`.

## Default or slim

clang-format's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a clang-format version such as `23.1.3`. `<minor>` and `<major>` are its shorter forms, `23.1` and `23`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

New versions come from the wheels on PyPI. A wheel has so far appeared between one day and a few weeks after the LLVM release, and some LLVM patch releases never got one: 22.1.6, 22.1.7, 21.1.3 and 21.1.4.

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current clang-format version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are built natively on GitHub-hosted runners, without emulation. The same `requirements.lock` is installed on all four combinations of distro and architecture: a glibc (`manylinux`) wheel on Ubuntu and a musl (`musllinux`) wheel on Alpine. The wheels are built by the wheel project, not here.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. `clang-format -i` and `git clang-format` write to the mounted files, and `git clang-format` also writes temporary files in `.git`, so run them as your own user:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

A check that only reads, such as `--dry-run`, can use a read-only mount (`-v "$PWD:/work:ro"`).

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` follow new clang-format releases and are rebuilt when the distro changes. The distro packages clang-format needs are listed in `/usr/local/share/randomcontainers/clang-format/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/clang-format:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends make \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache make`. The entrypoint is `["tini", "--", "clang-format"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new clang-format releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/clang-format:latest \
  --repo randomcontainers/clang-format --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists the distro and Python packages in it with their versions:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/clang-format:latest --format '{{ json .SBOM }}'
```

clang-format is installed from [`requirements.lock`](requirements.lock), which pins it to a version and the SHA-256 hashes of its wheels, and pip refuses anything else. The URL and hash of the installed wheel are in `/usr/local/share/randomcontainers/clang-format/source`, and the installed versions in `buildinfo` next to it. The wheels are built by the release workflow of [ssciwr/clang-format-wheel](https://github.com/ssciwr/clang-format-wheel) from LLVM's source release, which it downloads without checking a hash or signature, and a new version is used only when PyPI's provenance attestation shows it was published from that repository (see [Updates](#updates)).

## Updates

The project checks [clang-format on PyPI](https://pypi.org/project/clang-format/) every 15 minutes. A release is picked up once it is 24 hours old and PyPI holds a provenance attestation showing it was published from the [ssciwr/clang-format-wheel](https://github.com/ssciwr/clang-format-wheel) repository. The new version and a regenerated `requirements.lock` are then committed together and the images are rebuilt. A new LLVM major version is built only after the license files in [`licenses/bundled/`](licenses/bundled/) have been checked against it; until then its build stops and the existing tags stay as they are. Only the newest release is built; tags of older versions stay as they were last built. Wheel rebuilds with a fourth version part, such as 13.0.1.1, are not picked up.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags. After a new LLVM major version, new builds and these rebuilds pause until [`licenses/bundled/VERSIONS`](licenses/bundled/VERSIONS) has been rechecked and updated. The Alpine images pause the same way after a wheel whose `libstdc++` and `libgcc_s` come from a different GCC build.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim --build-arg VERSION=<version> -t clang-format:local .
```

For the Alpine image, use `Dockerfile.alpine` and also pass the two `extra-artifacts` entries of `package.yml` as build arguments: `GCC_VERSION` is the `version` of `gcc`, and `GCC_URL`, `GCC_SHA256`, `APORTS_URL` and `APORTS_SHA256` are the `url`, with `{version}` filled in, and the `sha256` of each entry. `<version>` must be the version pinned in `requirements.lock`; the build stops otherwise.

The command that produced `requirements.lock` is on its second line; it resolves for Python 3.14, the version on both bases. To regenerate the lock at the version in `package.yml`, run `rc lock requirements.lock` in this directory. `rc` is the command-line tool of [randomcontainers/ci](https://github.com/randomcontainers/ci), and it needs uv on `PATH`. The build also stops when the license files in [`licenses/bundled/`](licenses/bundled/) may not match the wheel: when the major version differs from the LLVM version in `licenses/bundled/VERSIONS`, when `clang_format.libs` holds a library other than `libstdc++` and `libgcc_s`, or when the wheel's auditwheel SBOM names a GCC build of those two other than the one in `VERSIONS`. On Alpine it also stops when the GCC build in `VERSIONS` is not a build of `GCC_VERSION`.

## Licenses

clang-format is part of the LLVM project and licensed under the [Apache License 2.0 with LLVM Exceptions](https://llvm.org/LICENSE.txt) (Apache-2.0 WITH LLVM-exception). The binary links LLVM's Support library statically, which contains code under other terms: xxHash (BSD-2-Clause), regular expression code from OpenBSD (BSD-3-Clause and ISC), Unicode conversion code and data (Unicode-DFS-2015 and Unicode-DFS-2016), BLAKE3 (CC0-1.0 or Apache-2.0, distributed here under Apache-2.0), and an MD5 implementation placed in the public domain. The wheel project's own files are under the Apache License 2.0, and pip, which stays in the virtual environment, includes packages under the Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, MIT, MPL-2.0 and PSF-2.0 licenses. On Alpine, the `libstdc++` and `libgcc_s` from the wheel are licensed under GPL-3.0-or-later WITH GCC-exception-3.1. The complete SPDX expression is in the image's `org.opencontainers.image.licenses` label, and each package's license files are in `/usr/local/share/randomcontainers/clang-format/licenses/`.

The wheels include only the wheel project's license, so this repository adds the others in [`licenses/bundled/`](licenses/bundled/): LLVM's and Clang's `LICENSE.TXT`, the notices of the third-party files in LLVM's Support library, and GCC's `COPYING3` and `COPYING.RUNTIME`. The image has them in `/usr/local/share/randomcontainers/clang-format/licenses/clang-format/bundled/`, the GCC texts only on Alpine. The `SOURCES` file there names the source releases they come from.

The `libstdc++` and `libgcc_s` in the musllinux wheels are taken from Alpine's libstdc++ and libgcc 14.2.0-r6 packages, which Alpine builds from GCC 14.2.0 with its own patches. Every release of this repository, named `v<version>` after the clang-format version, has the corresponding source attached: `gcc-14.2.0.tar.xz` with its signature, and Alpine's aports repository at the commit those packages were built from, as `<commit>.tar.gz`, with the build recipe and patches in `main/gcc`. Both are pinned under `extra-artifacts` in `package.yml`, and `SOURCES` explains how the libraries were matched to them. In the Alpine image, the last lines of `/usr/local/share/randomcontainers/clang-format/source` name the two archives and the release.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
