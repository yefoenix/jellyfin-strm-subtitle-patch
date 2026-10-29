# jellyfin-strm-subtitle-patch

Patched `Emby.Server.Implementations.dll` for **Jellyfin 10.11.11**, fixing external subtitles being dropped from `.strm` items on every playback.

- Upstream PR: [jellyfin/jellyfin#17790](https://github.com/jellyfin/jellyfin/pull/17790) (open, not merged)
- Upstream issues: [#15882](https://github.com/jellyfin/jellyfin/issues/15882), [#10338](https://github.com/jellyfin/jellyfin/issues/10338)

## The bug

`MediaSourceManager.GetPlaybackMediaSources` forces a `FullRefresh` with `EnableRemoteContentProbe` for every `.strm` item on **every** `PlaybackInfo` call. That refresh is handed the injected `IDirectoryService`, which is registered as a **singleton** in `ApplicationHost` and caches each folder listing for the life of the process.

`MediaInfoResolver.GetExternalFiles` therefore sees a stale listing of the item folder, so an external subtitle added after the folder was first listed (OpenSubtitles download, a newly dropped `.srt`/`.ass`) is not rediscovered, and `SaveMediaStreams` rewrites the stream list without it.

Symptom: the subtitle appears in the UI right after download, disappears on the next playback, and comes back after a server restart.

## The fix

One file, 6 insertions / 2 deletions — see [`patches/10.11.11-strm-fresh-directoryservice.patch`](patches/10.11.11-strm-fresh-directoryservice.patch).

For `.strm` items only, the playback-time refresh gets a **fresh** `DirectoryService` instead of the process-wide singleton, so remote probing works from the current folder contents. Probe behaviour is unchanged; non-`.strm` items keep using the injected service.

No public API changes — `MediaSourceManager` is an internal implementation class and the assembly version stays `10.11.11` (`SharedVersion.cs` at the tag).

## Prebuilt download

[Release `v10.11.11-fix.1`](https://github.com/yefoenix/jellyfin-strm-subtitle-patch/releases/tag/v10.11.11-fix.1) — `Emby.Server.Implementations.dll` (assembly version `10.11.11.0`), `.pdb`, and SHA-256.

```
d2bfdab55e240a5c5c58507e472db102868ca5cbfa44e253e5c6c6ef4ceaec8a  Emby.Server.Implementations.dll
```

## Build

GitHub Actions builds it on `windows-latest`: clone `jellyfin/jellyfin` at tag `v10.11.11`, apply the patch, `dotnet build Emby.Server.Implementations -c Release`, upload `Emby.Server.Implementations.dll` + `.pdb` + SHA-256 as artifacts.

Only this one project is built. The whole server is not rebuilt.

## Install (Windows binary install)

Default install path: `C:\Program Files\Jellyfin\Server\`.

1. Stop Jellyfin (tray icon → Quit, or `Stop-Service -Name JellyfinServer` when running as a service).
2. Back up the original:
   `copy "C:\Program Files\Jellyfin\Server\Emby.Server.Implementations.dll" "C:\Program Files\Jellyfin\Server\Emby.Server.Implementations.dll.orig"`
3. Replace `Emby.Server.Implementations.dll` with the artifact.
4. Start Jellyfin.

Rollback: restore `Emby.Server.Implementations.dll.orig`.

Verify after upgrade of Jellyfin to a newer version: this patch is pinned to 10.11.11 and must be rebuilt against the new tag.

## Scope / caveats

- This is a personal patch, not affiliated with the Jellyfin project. Upstream may fix the same symptom differently (see PRs #16869, #17050).
- `.strm` subtitles delivered as `DeliveryMethod=External`; no transcoding or burn-in behaviour is changed.

## License

Jellyfin is licensed under the **GNU GPL v2.0** — see [LICENSE](LICENSE). The patch here is a derivative work of Jellyfin and carries the same license.

Corresponding source for any binary built from this repository: `jellyfin/jellyfin` at tag `v10.11.11` plus the patch file in `patches/`.
