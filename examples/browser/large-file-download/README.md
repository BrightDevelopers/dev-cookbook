# Large File Download Example

![Runtime: Browser](https://img.shields.io/badge/runtime-browser-blue)
![BrightSign OS: 8.x | 9.x](https://img.shields.io/badge/BrightSign_OS-8.x_%7C_9.x-purple)

## Introduction

This example demonstrates how to download a large file (multi-GB) to a BrightSign player's local asset pool using the native `@brightsign/assetpoolfetcher` module. The fetcher streams to disk with bounded memory, honors the player-wide proxy automatically, and surfaces progress and error events for UI feedback.

The application is defined in `index.html` which uses the `assetpoolfetcher` Node module via `roHtmlWidget`. The `autorun.brs` file tells the player to run the application.

## Why Use `assetpoolfetcher`

`@brightsign/assetpoolfetcher` is BrightSign's native path for large asset downloads. It streams to disk with bounded memory, retries on transient failures, enforces a minimum transfer rate to detect stalled connections, and honors the player-wide proxy configured via `roNetworkConfiguration.SetProxy()` or the DWS. Partner apps do not need to configure the proxy per deployment.

The browser's Fetch API buffers the entire response body in memory, which causes an OOM crash on multi-GB files on memory-constrained players. Node's `http`/`https` modules can stream to disk via `stream.pipeline()`, but they do not inherit the player's OS-level proxy. `assetpoolfetcher` resolves both constraints natively.

## How It Works

1. **Asset pool and fetcher**: `index.html` creates an asset pool at a configured path, then creates a fetcher against that pool.
2. **No proxy params passed**: The fetcher honors the player-wide proxy automatically; the app code holds no proxy settings.
3. **Progress and file events**: The fetcher emits `progressevent` (per-file byte counts) and `fileevent` (per-file completion with response code and any error).
4. **Resolve local path**: After completion, `AssetPoolFiles.getPath()` returns the local path to the downloaded file.
5. **Diagnostic proxy read**: `networkstatus.getHostStatus()` returns the OS-configured proxy for logging only; the fetcher does not need it.

## Prerequisites

1. **Network connection**: The BrightSign player should be connected to a network with access to the file server.
2. **SD card**: The destination SD card should have enough free space. Format as exFAT to avoid FAT32 file-size limits on files larger than 4 GB.
3. **Optional proxy**: If the deployment network requires an HTTP proxy, configure it via `roNetworkConfiguration.SetProxy()` or the DWS. The sample will use it automatically.

## Configuration

Edit the constants at the top of the `<script>` section in `index.html`:

- `FILE_URL`: The URL of the file to download. HTTPS recommended.
- `ASSET_NAME`: The filename to store the asset under in the pool, for example `"largevideo.mp4"`.
- `POOL_PATH`: The on-player path for the asset pool. Default `"SD:/pool"`.

## Using a Proxy

Configure the proxy at the player level via `roNetworkConfiguration.SetProxy()` or the DWS. The sample reads nothing from app configuration for proxy; the player-wide setting applies automatically. HTTP and HTTPS targets are both supported.

For diagnostic visibility into which proxy the OS is currently using, the sample logs the result of `networkstatus.getHostStatus()` on startup. This is for logging only and does not affect fetcher behavior.

## Steps to Set Up and Run the Application

### 1. Manual Transfer

1. Copy the `autorun.brs` and `index.html` files to the root of the SD card.
2. Insert the SD card into the player.
3. Reboot the player.

### 2. Automated Transfer Using BrightSign CLI for DWS

Configure the CLI by choosing a name for your player and passing your player's information:

`bsc local player --add --player playerName --ip ip-address --user username --pass password --storage sd`

Push files to the player:

`bsc local file --upload --player playerName --file ./index.html --destination sd/`
`bsc local file --upload --player playerName --file ./autorun.brs --destination sd/`

## Troubleshooting

- **No progress shown**: Verify the player is connected to the network and the `FILE_URL` is reachable from the player.
- **`EFBIG: file too large` error**: The SD card is FAT32-formatted. Reformat as exFAT for files larger than 4 GB.
- **Download fails silently through a proxy**: Confirm the OS-level proxy is configured on the player via `roNetworkConfiguration.SetProxy()` or the DWS. Confirm `networkstatus.getHostStatus()` returns the expected proxy on the diagnostic log line printed at startup.
- **`fileevent` reports an error**: The error field surfaces on the per-file completion event. Check `e.responseCode` and the error string for cause.
