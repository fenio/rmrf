# GTFO

**G**et **T**he **F***** **O**ut - Reclaim disk space on GitHub runners.

GitHub-hosted runners come with a lot of pre-installed software you probably don't need. This action removes the bloat and optionally merges multiple disks into a single large volume.

## Features

- **Fast deletion** using [rmz](https://github.com/SUPERCILEX/fuc) (parallel, Rust-based rm)
- **Configurable cleanup** - choose what to remove
- **Disk merging** - combine root and `/mnt` into a single LVM volume (~100GB)
- **Btrfs support** - optional zstd compression for even more space

## Usage

### Basic (cleanup only)

```yaml
- uses: fenio/gtfo@v1
```

This removes ~20GB of bloat (Android SDK, .NET, Haskell, Boost, Swift, CodeQL).

### With disk merging

> **Important:** When using `merge-disks: true`, the action mounts a fresh filesystem at `/home/runner/work`. This means it **must run before `actions/checkout`**, or your checked-out code will be overwritten.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # GTFO must run FIRST when using merge-disks
      - uses: fenio/gtfo@v1
        with:
          merge-disks: 'true'
      
      # Now checkout into the merged workspace
      - uses: actions/checkout@v4
      
      # Your build steps...
```

Creates a ~100GB unified workspace by combining freed space with the `/mnt` disk.

### With Btrfs compression

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # GTFO must run FIRST when using merge-disks
      - uses: fenio/gtfo@v1
        with:
          merge-disks: 'true'
          use-btrfs: 'true'
      
      # Now checkout into the merged workspace
      - uses: actions/checkout@v4
```

Uses Btrfs with zstd compression for the merged volume.

### Custom configuration

```yaml
- uses: fenio/gtfo@v1
  with:
    remove-android: 'true'          # ~12GB
    remove-dotnet: 'true'           # ~2GB
    remove-haskell: 'true'          # ~2GB
    remove-boost: 'true'            # ~1GB
    remove-swift: 'true'            # ~1.5GB
    remove-codeql: 'true'           # ~1GB
    remove-hostedtoolcache: 'false' # ~8GB - Go, Node.js, Python, Ruby, etc.
    remove-docker-images: 'false'   # ~4GB
    merge-disks: 'false'
    use-btrfs: 'false'
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `remove-android` | Remove Android SDK (~12GB) | `true` |
| `remove-dotnet` | Remove .NET SDK (~2GB) | `true` |
| `remove-haskell` | Remove GHC/Haskell (~2GB) | `true` |
| `remove-boost` | Remove Boost (~1GB) | `true` |
| `remove-swift` | Remove Swift (~1.5GB) | `true` |
| `remove-codeql` | Remove CodeQL (~1GB) | `true` |
| `remove-hostedtoolcache` | Remove cached tool versions - Go, Node.js, Python, Ruby, etc. (~8GB) | `false` |
| `remove-docker-images` | Remove Docker images (~4GB) | `false` |
| `merge-disks` | Merge root and /mnt into single LVM volume | `false` |
| `use-btrfs` | Use Btrfs with zstd compression (requires merge-disks) | `false` |

## How it works

### Cleanup mode (default)

1. Reports initial disk usage
2. Downloads `rmz` for fast parallel deletion
3. Removes selected bloat directories
4. Reports final disk usage

### Merge mode

> **Note:** The disk merging feature is somewhat hackish - it relies on undocumented details of GitHub runner disk layout that could change without notice. It works today, but use at your own risk.

1. Removes bloat (as above)
2. Disables swap and unmounts `/mnt`
3. Creates a loopback file from freed space on root (~80% of available)
4. Creates LVM volume group combining the former `/mnt` disk and loopback
5. Formats as ext4 or Btrfs (with zstd compression)
6. Mounts at `/home/runner/work`

### Space gains

| Mode | Approximate free space |
|------|----------------------|
| Cleanup only | ~35-40GB on root |
| Merge (ext4) | ~100GB unified |
| Merge (Btrfs+zstd) | ~100GB+ (compression dependent) |

## Bloat locations on GitHub runners

| Path | Size | Removed by |
|------|------|------------|
| `/usr/local/lib/android` | ~12GB | `remove-android` |
| `/opt/hostedtoolcache` | ~8GB | `remove-hostedtoolcache` |
| `/usr/share/dotnet` | ~2GB | `remove-dotnet` |
| `/opt/ghc` | ~2GB | `remove-haskell` |
| `/usr/local/.ghcup` | ~1GB | `remove-haskell` |
| `/usr/local/share/boost` | ~1GB | `remove-boost` |
| `/usr/share/swift` | ~1.5GB | `remove-swift` |
| `/opt/hostedtoolcache/CodeQL` | ~1GB | `remove-codeql` |
| Docker images | ~4GB | `remove-docker-images` |

## License

MIT
