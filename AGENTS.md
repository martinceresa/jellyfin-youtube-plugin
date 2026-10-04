# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

**YouTubeSync** is a Jellyfin server plugin (C#, .NET 10, Jellyfin 12.1). It turns YouTube channels and playlists into a Jellyfin library without downloading the videos:

- **Sync**: a scheduled task calls `yt-dlp` and writes a folder tree of `.strm` + `.nfo` files and artwork into a configured library path.
- **Playback**: each `.strm` file contains `{JellyfinBaseUrl}/YouTubeSync/resolve/{videoId}`. When played, the plugin resolves the video with `yt-dlp` and either redirects (302) to the YouTube CDN URL or serves a local ffmpeg HLS session.

Runtime dependencies on the server: `yt-dlp` (required), `ffmpeg` (only for managed transcoding).

## Repository layout

```
Jellyfin.Plugin.YouTubeSync/
  Plugin.cs                      Plugin entry point, config page registration, DI registration (PluginServiceRegistrator)
  meta.json                      Plugin metadata (guid, version, targetAbi); version is patched by CI
  Configuration/
    PluginConfiguration.cs       All user settings (persisted by Jellyfin as XML)
    SourceDefinition.cs          A channel/playlist source; builds the yt-dlp URL (channel feed tab suffixes)
  Controllers/
    YouTubeSyncController.cs     HTTP API: /YouTubeSync/resolve/{id}, /session/{id}/{file}, /source-info
  Services/
    YtDlpService.cs              Every yt-dlp invocation: format selectors, JSON parsing, thumbnail scoring, date parsing
    SimpleResolveCache.cs        In-memory TTL cache of resolved playback URLs
  Playback/
    ResolveService.cs            Video ID -> playback URL, via the cache
    ManagedTranscodeService.cs   ffmpeg -> HLS sessions in temp dir, hardware accel modes, idle cleanup
  Sync/
    SyncTask.cs                  IScheduledTask "YouTube Sync" (every 6 h by default)
    SyncService.cs               Main sync: fetch entries, retention, write files, clean up obsolete content
    SyncPlaylistFeedExpander.cs  Channel "Playlists" feed: each playlist becomes a season
    SyncSeasonLayout.cs          Season/episode numbering, folder naming, filename sanitising
    SyncNfoBuilder.cs            Kodi-style NFO XML (tvshow, season, episode, movie)
    SyncArtworkHelper.cs         Artwork download with YouTube thumbnail fallback URLs
  Metadata/                      DTOs (VideoMetadata, SourceInfo)
  Web/
    youtubeSyncConfig.html/.js   Dashboard config page (embedded resources)
local-testing/                   Docker-based disposable Jellyfin for smoke tests (PowerShell scripts)
manifest.json                    Jellyfin plugin repository manifest (updated by CI on release)
.github/workflows/release.yml    Tag-triggered release pipeline
```

## Build

```bash
dotnet publish Jellyfin.Plugin.YouTubeSync/Jellyfin.Plugin.YouTubeSync.csproj \
  -c Release --no-self-contained -o publish/
```

- Needs the .NET 10 SDK. Jellyfin packages are referenced with `ExcludeAssets=runtime`, so only `Jellyfin.Plugin.YouTubeSync.dll` + `meta.json` are deployed.
- There is **no test project and no linter config**. Check your changes by building, and for behavioural changes run the smoke checklist in `local-testing/README.md`.
- Build outputs (`bin/`, `obj/`, `publish/`) and local test state (`.local/`) are git-ignored.

## Local testing

`local-testing/` runs Jellyfin 12.1 in Docker with `yt-dlp`, `ffmpeg` and `deno` installed, on `http://localhost:8096`. Its state lives in `.local/jellyfin/`.

The helper scripts are PowerShell (`publish-local-plugin.ps1`, `start-…`, `stop-…`, `reset-…`, `bootstrap-…`). Without `pwsh` you can do the same by hand: publish into `.local/build/publish`, copy the DLL and `meta.json` to `.local/jellyfin/plugins/YouTubeSync/`, then run `docker compose -f local-testing/docker-compose.yml up --build -d`.

See `local-testing/README.md` for the smoke checklist to run before releasing changes that touch sync, metadata, sources or playback.

## Release process

Pushing a tag `vX.Y.Z` triggers `.github/workflows/release.yml`. The workflow:

1. Sets `meta.json` version to `X.Y.Z.0` and builds with that assembly version.
2. Zips the DLL and `meta.json` as `jellyfin-youtubesync_X.Y.Z.zip`, computes its MD5, and creates a GitHub release.
3. Adds a new entry at the top of `manifest.json` and commits it to `main` as `chore: update manifest.json for vX.Y.Z [skip ci]`.

Do not hand-edit `manifest.json` version entries or bump `meta.json` for a release; CI does both. If you change the targeted Jellyfin version, update it everywhere it appears: the csproj target framework and package versions, `meta.json` `targetAbi`/`framework`, the hard-coded `targetAbi` and `dotnet-version` in `release.yml`, the `local-testing/Dockerfile` base image, `README.md`, `local-testing/README.md` and this file. `targetAbi` is the minimum server version (Jellyfin loads a plugin when server version ≥ `targetAbi`); follow the official plugins and set it to the Jellyfin version the packages are built against (`12.1.0.0`). The manifest was reset when the project moved to this fork, so it only lists Jellyfin 12 builds; there is no 10.11 line here.

## Conventions and invariants

- **Plugin GUID** `55a3502b-b6b2-4a3c-93d7-f3c4e7b1e0d5` appears in `Plugin.cs`, `meta.json`, `manifest.json` and `Web/youtubeSyncConfig.js` (`pluginUniqueId`). They must always match.
- **Config page wiring**: `Plugin.GetPages()` registers `youtubeSyncConfig` and `youtubeSyncConfigjs`. The HTML loads the JS through `data-controller="__plugin/youtubeSyncConfigjs"`. Any new web asset must be added both as an `<EmbeddedResource>` in the csproj and in `GetPages()`.
- **Adding a setting**: add a property with a default to `PluginConfiguration`, then read and write it in `youtubeSyncConfig.js` (load/save handlers) and add a field to the HTML. Do not rename or remove existing properties, because saved user configs are keyed by property name. `MaxVideosPerSource` is labelled legacy but `SyncService` still uses it to cap entry scans.
- **Adding a service**: register it as a singleton in `PluginServiceRegistrator.RegisterServices`.
- **Endpoint authorization**: Jellyfin 12 disables legacy authorization (`X-Emby-Token`, `X-Emby-Authorization`, `api_key`) by default. Dashboard JS must call the server through `ApiClient.getJSON`/`ApiClient.fetch` (which send `Authorization: MediaBrowser … Token="…"`), never raw `fetch` with a token header. `/YouTubeSync/source-info` is `[Authorize(Policy = Policies.RequiresElevation)]`; `/resolve` and `/session` are `[AllowAnonymous]` because .strm playback requests carry no token.
- **Outbound HTTP** (artwork downloads) uses Jellyfin's `IHttpClientFactory` with `NamedClient.Default`, not a hand-rolled `HttpClient`.
- **Accessing config**: services read `Plugin.Instance?.Configuration` at call time and do not inject it, so changes made in the dashboard take effect without a restart.
- **yt-dlp calls** go through `YtDlpService`. Pass arguments with `ProcessStartInfo.ArgumentList`, never a concatenated string. On failure, log and return `null` instead of throwing.
- **File writes during sync** use `WriteTextFileIfChangedAsync`, which skips identical content to avoid triggering Jellyfin rescans. Artwork is only downloaded when it is missing.
- **On-disk layout** (series mode): `{LibraryBasePath}/{Source Name}/Season {YYYY}/{Title} [{videoId}]/{Title}.strm|.nfo|poster.*`. For the channel "Playlists" feed, seasons are `Season NN` (one per playlist). In movies mode there are no season folders. `CleanupObsoleteContent` deletes video folders that are no longer wanted, so changes to folder naming affect existing user libraries.
- **Naming**: the UI says "Simple"/"Enhanced" playback. In code these are the direct resolve path and "managed transcoding" (`AllowManagedTranscoding`, `ManagedTranscodeService`). Managed transcoding must always fall back to the direct resolve path when it cannot start.
- **Style**: file-scoped namespaces, nullable enabled, `ConfigureAwait(false)` on awaits, `_camelCase` private fields, XML doc comments on public members, structured logging templates (`{VideoId}`, not string interpolation).
- **Commits**: conventional prefixes (`feat:`, `fix:`, `chore:`).
