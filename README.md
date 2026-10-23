# minimap2

Container images with [Minimap2](https://github.com/lh3/minimap2), which aligns DNA and mRNA sequences to a reference genome: PacBio and Oxford Nanopore reads, assembly contigs, spliced RNA reads and short reads. The default image adds [Samtools](https://github.com/randomcontainers/samtools) for sorting and indexing the alignments. Minimap2 is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when a new Minimap2 release comes out and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Minimap2 developers. Report problems with the image in this repository and Minimap2 bugs at [lh3/minimap2](https://github.com/lh3/minimap2/issues).

## Quick start

Map Oxford Nanopore reads in the current directory to a reference and write SAM:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 -ax map-ont -t 8 -o aln.sam ref.fa reads.fq.gz
```

The entrypoint runs `minimap2` under `tini`, so its options and files go after the image name. `-x` picks the preset for the kind of sequences, `-a` writes SAM instead of PAF, and `-t` sets the number of threads. A few more commands:

```sh
# Save the index of a reference once, then map against the index
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 \
  -x map-ont -d ref.mmi ref.fa
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 \
  -ax map-ont -t 8 -o aln.sam ref.mmi reads.fq.gz

# Align an assembly to a reference and write PAF with the cs tag
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 \
  -cx asm5 --cs -o asm.paf ref.fa asm.fa

# Align spliced long reads from RNA
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 \
  -ax splice -o rna.sam ref.fa rna.fq.gz

# Map paired-end short reads
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/minimap2 \
  -ax sr -t 8 -o aln.sam ref.fa reads_1.fq.gz reads_2.fq.gz
```

Running the image without arguments prints the options and the presets. The [Minimap2 manual](https://lh3.github.io/minimap2/minimap2.html) describes each of them. The indexing options (`-k`, `-w`, `-H` and `-I`) are saved in the index and cannot be changed when mapping against it, so build the index with the preset you map with.

## What is in the image

- `minimap2` in `/usr/local/bin`. Apart from the distro's C library it loads only zlib.
- On `linux/amd64` the alignment code is compiled for SSE2 and for SSE4.1, and Minimap2 uses the SSE4.1 code when the CPU has it. On `linux/arm64` it is compiled for NEON.

Not included:

- `paftools.js` and the other scripts in Minimap2's `misc/` directory, which need the k8 JavaScript shell.
- `sdust`, `minimap2-lite`, the `libminimap2.a` library and its headers, the mappy Python module and the man page.

`/usr/local/share/randomcontainers/minimap2/buildinfo` lists the make command and the compiler flags.

## Default or slim

Use the default image (`latest`) for general work. It is the Samtools slim image with Minimap2 added, so one container can align reads and sort and index the alignments:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh ghcr.io/randomcontainers/minimap2 \
  -c 'minimap2 -ax map-hifi -t 8 ref.fa reads.fq.gz | samtools sort -o aln.bam --write-index -'
```

`slim` has only `minimap2` and what it needs. Use it as the base for your own image, or when you only write SAM or PAF.

The default image has every program of the Samtools slim image, including `bgzip`, `tabix` and `htsfile`; the [samtools README](https://github.com/randomcontainers/samtools#what-is-in-the-image) lists them. It has no BCFtools, which only the Samtools default image adds. The default image is also published as `ghcr.io/randomcontainers/minimap2-samtools`, built in the [minimap2-samtools](https://github.com/randomcontainers/minimap2-samtools) repository with the same contents and a different digest.

## Tags

`<version>` is a Minimap2 release such as `2.31`. `<minor>` and `<major>` are its shorter forms, `2.31` and `2`, and follow the newest release in that series. Minimap2 releases rarely have a point number, so `<version>` and `<minor>` are usually the same tag.

| Default (with Samtools) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Minimap2 version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Minimap2 writes the alignments to standard output unless `-o` names a file, and `-d` writes the index to the file it names. Reads and references can be plain or gzip-compressed FASTA or FASTQ.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no Samtools, so Samtools updates do not rebuild it. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Minimap2 release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/minimap2:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends procps \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
ENTRYPOINT []
```

This prepares the image for Nextflow, which needs `bash` and `ps` in a task's container and starts tasks with its own `/bin/bash` command, so the example clears the `minimap2` entrypoint. On Alpine, start from `slim-alpine` and use `apk add --no-cache bash procps`. Leave out `ENTRYPOINT []` to keep the slim image's entrypoint, `["tini", "--", "minimap2"]`. For a pipeline step that pipes Minimap2 into Samtools, start from the default image instead, `ubuntu` or `alpine` with a digest. To pick up new Minimap2 releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/minimap2/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages Minimap2 needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/minimap2:latest \
  --repo randomcontainers/minimap2 --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/minimap2:latest --format '{{ json .SBOM }}'
```

The SBOM does not list Minimap2 itself, because it is compiled from source; `minimap2 --version` and `buildinfo` name its version. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`. Minimap2 has no test suite, so after compiling the build aligns the two mitochondrial genomes in the tarball's `test/` directory, and the image tests check the alignments of generated reads before an image is published.

## Updates

The project checks the releases of [lh3/minimap2](https://github.com/lh3/minimap2/releases) every 15 minutes. A release is picked up once it is 24 hours old. The Minimap2 developers publish no checksum file or signature, so the tarball is checked against the SHA-256 digest GitHub records for the release asset. The new version and the tarball's SHA-256 are then committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Samtools image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t minimap2:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Minimap2 is under the MIT license, from the Dana-Farber Cancer Institute and the Broad Institute. The klib headers compiled into it (`khash.h`, `kseq.h`, `ksort.h`, `kvec.h` and `krmq.h`, by Attractive Chaos) and the sse2neon header that the arm64 build uses for its NEON code are MIT too. The amd64 build also compiles `ksw2_dispatch.c`, whose few lines of `cpuid` inline assembly are adapted from Intel's linux-sgx, which is BSD-3-Clause. Minimap2 ships no Intel notice for them, so this repository has the copyright notice and license from linux-sgx in [`licenses/LICENSE.linux-sgx`](licenses/LICENSE.linux-sgx).

`/usr/local/share/randomcontainers/minimap2/licenses/` holds Minimap2's `LICENSE.txt`, `NOTICES`, `LICENSE.sse2neon` and `LICENSE.linux-sgx`. The build fills `NOTICES` with the first copyright comment of each C file: the klib MIT notices, and the PostgreSQL notice in `misc.c`, which covers a Windows replacement for `gettimeofday` that is not compiled on Linux. `LICENSE.sse2neon` has the sse2neon contributors and license, cut from the header; every image has it, although only the arm64 build compiles the header in. Every image also has `LICENSE.linux-sgx`, although only the amd64 build compiles the code it covers. The source tarball URL is in `/usr/local/share/randomcontainers/minimap2/source`. The image's license label is `MIT AND BSD-3-Clause`; the default image adds Samtools' terms, listed in its [README](https://github.com/randomcontainers/samtools#licenses). The Ubuntu and Alpine packages in the image keep their own licenses, for example the zlib license for zlib. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
