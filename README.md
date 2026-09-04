## AIOZ DePIN Worker

Official DePIN worker node implementation of the AIOZ Network.

## What is the AIOZ DePIN Worker?

The AIOZ DePIN Worker connects your machine to the AIOZ DePIN storage network. It stores encrypted
pieces of user data, serves them back on demand, and earns rewards for the capacity and bandwidth
it contributes.

A worker node is made of three programs, all published here:

| Binary | What it does |
| --- | --- |
| `worker` | the node itself; `worker start` runs it |
| `worker-updater` | optional supervisor that keeps `worker` up to date |
| `keytool` | mints and manages the node identity and its wallet key |

## Requirements

- Windows 10 version 22H2 64-bit or above
- Ubuntu 14.04 64-bit or above (amd64 or arm64)
- macOS Catalina version 10.15 or above (Apple Silicon or Intel)
- Disk space you are willing to allocate

No public IP address or port forwarding is required. A node behind NAT reaches the network
through AIOZ relays, and will opportunistically try a UPnP/NAT-PMP port mapping to become
directly dialable where the router allows it.

## Downloads

Each release publishes one archive per platform holding all three programs:

```
aioz-depin-worker_<os>_<arch>.zip
```

| Platform | Suffix |
| --- | --- |
| Linux x86_64 | `linux_amd64` |
| Linux ARM64 | `linux_arm64` |
| macOS Apple Silicon | `darwin_arm64` |
| macOS Intel | `darwin_amd64` |
| Windows x86_64 | `windows_amd64` |

The programs are also published individually as `<program>_<os>_<arch>.zip` if you only need
one of them. Both are covered by the `sha256sums` manifest and its signature `sha256sums.sig`.

## Getting started

### Linux and macOS

Substitute your platform suffix for `linux_amd64`.

```shell
BASE=https://github.com/AIOZNetwork/aioz-depin-worker/releases/download/v0.1.0
curl -LO $BASE/aioz-depin-worker_linux_amd64.zip
curl -LO $BASE/sha256sums
```

Verify what you downloaded before running it.

```shell
sha256sum -c sha256sums --ignore-missing
```

Extract it. The archive is flat, so this drops `worker`, `worker-updater` and `keytool` into
the current directory.

```shell
unzip -o aioz-depin-worker_linux_amd64.zip
chmod +x worker worker-updater keytool
```

Verify the installation.

```shell
./worker version
```

The output should be the version of the AIOZ DePIN Worker.

### Windows

The scripts below are written for Windows PowerShell.

```shell
$base = "https://github.com/AIOZNetwork/aioz-depin-worker/releases/download/v0.1.0"
curl.exe -LO "$base/aioz-depin-worker_windows_amd64.zip"
curl.exe -LO "$base/sha256sums"
Expand-Archive -Path aioz-depin-worker_windows_amd64.zip -DestinationPath .
```

Verify the installation.

```shell
.\worker.exe version
```

### Verifying the signature

`sha256sums` is signed with [cosign](https://github.com/sigstore/cosign). If you have the AIOZ
release public key, you can check that the manifest itself was not tampered with.

```shell
curl -LO $BASE/sha256sums.sig
cosign verify-blob --key aioz-release.pub --signature sha256sums.sig sha256sums
```

## Setting up a node

### 1. Create the node identity

The identity is both the node's TLS credential on the network and the wallet that receives its
rewards. Creating one mines a node ID to a target difficulty, which takes a few minutes.

```shell
./keytool create worker --identity-dir ./identity --difficulty 36
```

This writes `ca.cert`, `ca.key`, `ca.nonce`, `identity.cert`, `identity.key` and `priv_key.json`
into `./identity/worker`, and prints the mnemonic phrase and the CA nonce.

Pass `--identity-dir` explicitly, as above, and use the same path everywhere. `keytool` and
`worker` do not default to the same location, so relying on the defaults leaves the node unable to
find the identity you just made.

**IMPORTANT NOTE**

- Treat your `priv_key.json`, `identity.key` and mnemonic phrase as **wallet secrets**. Anyone who
  has them can **control your node rewards**.
- Write down the **mnemonic phrase and the nonce**. Together they recreate the exact same identity
  without re-mining it, via `keytool create worker --mnemonic-file <file> --nonce <n>`.
- Use a **dedicated key for each node**; **do not reuse** the key from your main wallet, exchange
  account, or other nodes. Running two nodes from one key **will get them disqualified**. The node
  ID is derived from the key, so both machines present the same identity to the network, only one
  of them holds any given piece, and the audits that land on the other one fail. Enough audit
  failures disqualify the node, and **disqualification is permanent** - the node is out for good,
  along with the rewards it had not yet withdrawn.
- Store backups in an **offline, encrypted location** such as encrypted USB drives, air-gapped
  devices, or hardware security modules (HSM).
- **Never paste** your mnemonic or private key into websites, chats, or support tickets.

### 2. Write the config

```shell
./worker setup --identity-dir ./identity/worker
```

This creates the configuration directory and writes `config.yaml` into it:

| OS | Configuration directory |
| --- | --- |
| Linux | `$XDG_DATA_HOME/aioznode`, or `~/.local/share/aioznode` |
| macOS | `~/Library/Application Support/Aioznode` |
| Windows | `%AppData%\Aioznode` |

Both the configuration and the identity directory can be moved with the global `--config-dir` and
`--identity-dir` flags, which every command accepts.

Edit `config.yaml` to set at least:

- `trust-sources` - the coordinator to join, as `<coordNodeID>@<host>:7777`
- `storage.path` - where pieces are stored
- `storage.allocated-disk-space` - how much you are contributing, for example `500GB`

Leave the networking settings alone unless you know you need them. `server.have-public-address`
defaults to `false`, which is the behind-NAT path: the node reserves a circuit on a relay and is
reachable through it. Set it to `true`, and set `contact.external-address`, only if the machine
really does have a public address other nodes can dial directly.

Every setting can also be given as an environment variable with the `AIOZ_WORKER_` prefix, which
takes precedence over `config.yaml`. For example `AIOZ_WORKER_STORAGE_ALLOCATED_DISK_SPACE=500GB`.

### 3. Run the node

```shell
./worker start --identity-dir ./identity/worker
```

Leave it running. On Windows, run it as a Windows Service so it starts at boot.

## Usage

### View what the node has earned

```shell
./worker balance
```

Add `--json` for the raw response, and `--coord <nodeID>` if your node trusts more than one
coordinator.

### Withdraw rewards

```shell
./worker withdraw 0x9A20600a143745404f1AA1C69Bd280f4bD5B4408 --amount 12.5
```

`--amount` is in **AIOZ**, the same unit `worker balance` reports, so you can withdraw a figure you
actually saw. Omit it to withdraw everything available. The coordinator records the request and an
operator sends the funds. Only one request may be pending at a time - `worker balance` shows a
pending one.

Useful flags: `--dry-run` to see what would be requested without requesting it, `-y` to skip the
confirmation prompt.

### Recover a key from a mnemonic phrase

```shell
./keytool recover --mnemonic-file mnemonic.txt
```

The phrase is read from a file or from standard input, never from a command-line argument, so that
it cannot leak through your shell history or the process list.

## Automatic updates

`worker-updater` runs alongside the node, polls the AIOZ version control service, and replaces the
`worker` binary in place when a new release is rolled out to your node.

```shell
./worker-updater run \
  --binary-location ./worker \
  --identity-dir ./identity/worker \
  --version.server-address https://<versioncontrol-host> \
  --version.check-interval 15m
```

The updater signs its requests with the node identity, so it needs the same `--identity-dir` the
node uses.

Rollouts are staged, so a new release does not reach every node at once. Your node updates when the
rollout cursor reaches it.

Notes:

- Because the updater replaces the executable file on disk, the `worker` binary and its directory
  must be **writable** by the account the updater runs as.
- The node is stopped and restarted to apply an update. Run it as a Windows Service, a systemd
  unit, or under a supervisor so it comes back automatically.
- Automatic updating is still **experimental**. If you would rather control upgrades yourself, do
  not run `worker-updater`: download the new zip from this repository and replace the binary.

## License

Apache License 2.0. See [LICENSE](LICENSE).
