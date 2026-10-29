# Horizon Server

A standalone Minecraft: Java Edition 26.3 server written in Rust. Download it here if you only want an
optimized server. Vanilla Java clients and [Horizon](https://github.com/horizon-rust/horizon) clients can both
join. Gameplay follows vanilla 26.3: Horizon checks it against the real game with more than 300 parity tests.

## Where it comes from

This repo holds no code. The server is built from
[horizon-rust/horizon](https://github.com/horizon-rust/horizon), the same code that runs Horizon's single
player and LAN worlds:

- **the engine**: `horizon-engine`, the game run headless (`horizon-client --dedicated`), built without sound
  or controller support;
- **the supervisor**: `horizon-server`, the program you run. It covers `server.toml`, the console, the web panel,
  backups, updates, crash restarts, and a control API on 127.0.0.1.

Report bugs and send pull requests to [horizon-rust/horizon](https://github.com/horizon-rust/horizon/issues).
How the server is getting faster, measured against vanilla and Pumpkin:
[docs/server-next.md](https://github.com/horizon-rust/horizon/blob/main/docs/server-next.md).

## Downloads

**Planned:** this repo's releases, one archive per OS. Each one is made from a Horizon release tag
([#3974](https://github.com/horizon-rust/horizon/issues/3974)). The release workflow:

1. runs when horizon-rust/horizon publishes a release;
2. downloads that release's server archives and checks them against its signed `SHA256SUMS`;
3. publishes them here unchanged, under the same tag, with the same checksums.

Nothing is rebuilt here, so a download from either repo is byte for byte the same.

**Until then**, use Horizon's own releases:
- Linux: `horizon-server-linux-x86_64.tar.gz`.
- macOS and Windows: the game archive, which also holds `horizon-server`.

Or use the installer:

```sh
curl -fsSL https://raw.githubusercontent.com/horizon-rust/horizon/main/packaging/server/install.sh | sh
```

On Windows: `irm https://raw.githubusercontent.com/horizon-rust/horizon/main/packaging/server/install.ps1 | iex`.

## Game files

The server needs the vanilla 26.3 data, prepared from **your own** copy of Minecraft. Nothing of Mojang's is
redistributed here, and nothing is downloaded for you. Steps:
[docs/hosting.md](https://github.com/horizon-rust/horizon/blob/main/docs/hosting.md).

## Accounts

`online_mode = true` is the default, as on every Minecraft server: each player's account is checked with
Mojang's services. You can turn it off, as on vanilla's server, but an offline server checks no accounts.
Anyone who reaches it can join under any name, so keep it on your LAN or a tailnet. More in
[docs/hosting.md](https://github.com/horizon-rust/horizon/blob/main/docs/hosting.md), "Before you start".

## License

MIT, like Horizon. Minecraft is a trademark of Mojang Studios. This project is not affiliated with Mojang or
Microsoft.
