# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Go CLI (single `main` package, all files at repo root) that downloads anime from Crunchyroll, decrypts the Widevine DRM, and muxes video/audio/subtitles into an MKV via ffmpeg. It requires a Crunchyroll Premium account for premium content and a Widevine device (`.wvd` file, or `client_id.bin` + `private_key.pem`) to decrypt streams.

## Commands

```shell
go build .                                   # build the binary
go test -race -count=2 -timeout 5m ./...     # full test suite (matches CI; -count=2 catches concurrency/ordering bugs)
go test -race -run TestName ./...            # run a single test
go vet ./...
gofmt -l .                                   # CI fails if this lists any file; run `gofmt -w .` to fix
```

CI (`.github/workflows/tests.yml`) runs tests, a cross-platform build (linux/darwin/windows), and `go vet` + `gofmt -l` as separate jobs on every push/PR. Releases (`.github/workflows/release.yml`) build and attach binaries (`crdl-linux`, `crdl-macos-arm64`, `crdl-macos-x64`, `crdl-windows.exe`) on GitHub release publish.

Running the tool for real requires `-etp-rt` (a Crunchyroll session cookie) and either a `.wvd` file or `client_id.bin`/`private_key.pem` in the working directory — see README.md for the full flag reference and usage examples.

## Architecture

**Pipeline:** URL → resolve episode/season metadata (Crunchyroll `content/v2/cms` API) → fetch playback manifest (DASH MPD) → extract PSSH → get Widevine license/keys → download+decrypt video/audio/subtitle tracks concurrently → mux into MKV with ffmpeg.

**File responsibilities:**
- `main.go` — flag parsing, URL parsing (`/watch/` vs `/series/`), orchestrates single-URL or batch (`-file`) downloads.
- `token.go` — exchanges the `etp_rt` cookie for a bearer access token.
- `http_request.go` — `DoRequest` wraps `http.Client.Do` and transparently refetches the access token + retries once on a 401.
- `episode.go` — playback API (`getEpisode`, returns manifest URL/token/subtitle+caption lists) and CMS metadata API (`getEpisodeInfo`, `deleteStream` to release a playback session).
- `season.go` — CMS season/episode-list APIs.
- `mpd.go` — fetches and parses the DASH MPD (segmented shape), representation selection by quality (`getBaseUrl`), DASH `SegmentTimeline` expansion.
- `ondemand.go` — handles the *other* manifest shape Crunchyroll serves: single-file DASH with `SegmentBase` byte ranges instead of `SegmentTemplate`. `go-mpd` doesn't model this, so it's parsed manually (`onDemandMPD`) and downloaded via HTTP range requests instead of per-segment fetches. `isOnDemand(manifest)` decides which code path (`mpd.go` vs `ondemand.go`) a given track uses — both shapes are handled everywhere a manifest is downloaded.
- `drm.go` — Widevine: locates the PSSH in the MPD (handles Crunchyroll's PlayReady-tagged-but-actually-Widevine PSSH quirk via `toWidevinePssh`), gets a license via `gowidevine`, and decrypts fragmented MP4 by matching each track's KID to the correct content key (important: a license can carry multiple keys, one per track — do not assume the first key is right).
- `download.go` — the concurrency core. `streamSegments` fetches segments in parallel (`maxWorkers = 10`) but writes them to disk strictly in order with bounded buffering (`maxBufferedSegments`), so peak memory doesn't scale with title length. `downloadEpisode` fans out per audio-locale "version" (each dub is a separate playback session/token/manifest/license) plus subtitles, all concurrently, then joins and calls into `output.go` to mux. Supports `-audio-lang all` / `-subs-lang all` / `-cc-lang all` to auto-expand to every available locale.
- `progress.go` — multi-line terminal progress bar renderer; concurrent downloads each get their own line, redrawn in place.
- `output.go` — builds the `ffmpeg` argument list to mux video+audio+subs into one MKV: explicit `-map` per stream, VTT subs transcoded to SRT (ASS kept as `copy`), language/title metadata per track, disposition flags (first audio and first non-CC subtitle marked default), then cleans up temp files.
- `utils.go` — locale name/ISO-639-2 code tables, `sanitizeFilename`.

**Concurrency model to keep in mind when editing `download.go`:** each audio dub is an independent unit of work (own token, manifest, PSSH, license keys) and downloads in parallel with the others and with subtitles; only the video track is shared/downloaded once (identical across dubs) and races with the *first* audio version's download since they share that version's keys. Playback session tokens opened during a download are tracked (`activeStreams`) and always released via `deleteStream` in a deferred cleanup, including on panic.

**Test files** (`*_test.go`) sit next to the code they cover and exercise the pure/logic-heavy pieces (URL/lang parsing, timeline expansion, filename sanitizing, segment streaming ordering, on-demand byte-range parsing) rather than hitting the network.
