---
name: teams-custom-audio
description: Replace notification sounds and ringtones in New Teams Desktop.
---

# Teams Custom Audio

Teams stores MP3 audio files in its local WebView2 HTTP cache.

## Cache location

%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\WV2Profile_<profile>\Cache\Cache_Data

- Profile varies by account: `tfw` (work) or `tfl` (personal). Find it with `Cache_Data` dir search.
- `data_1` is locked while Teams runs: copy it to a temp folder and read the copy.

## Cache structure

- `data_1` contains resource URLs, file sizes and storage addresses.
- `f_XXXXXX` files contain raw audio data without file extensions.
- Cache entries are 256 bytes, starting at offset 8192.
- Key length: entry offset + 32 (Int32). Key (URL): entry offset + 96.
- Audio size: entry offset + 44 (Int32).
- Audio address: entry offset + 60 (UInt32).
- External address `0x80000121` maps to `f_000121` (bit 31 set, file type bits 28-30 = 0).
- A URL can appear in several entries; use the one with non-zero size (key ends with `:<hash>:0`).

## Audio replacement procedure

1. Stop `ms-teams.exe` (`taskkill /IM ms-teams.exe /F`).
2. Locate the requested audio URL in `data_1`.
3. Resolve its physical cache file and expected size.
4. Back up the original file.
5. Convert the custom audio to MP3 slightly under target size, e.g.
   `ffmpeg -i in.mp3 -t 18 -af afade=t=out:st=17:d=1 -map_metadata -1 -ar 48000 -ac 2 -b:a 128k -write_id3v2 0 raw.mp3`
6. Pad to the exact size: prepend an ID3v2.3 header (`ID3\x03\x00\x00` + syncsafe size) followed by zero bytes.
7. Replace the cache file without changing its name.
8. Restart Teams (`explorer.exe shell:AppsFolder\MSTeams_8wekyb3d8bbwe!MSTeams`) and verify playback.

## Known resources

- Incoming call: `Teams_Call_Ringing.mp3` + `ring.mp3` (identical, 304353 bytes, 7.6 s @ 320 kbps). Replace both.
- Default notification: `00_teams_basic_notification_r4_default.mp3`
- Ping: `a8_teams_basic_notification_r4_ping.mp3`

Base URL (domain varies by profile):
https://teams.public.onecdn.static.microsoft/evergreen-assets/audio/ (work)
https://tflteams.public.onecdn.static.microsoft/evergreen-assets/audio/ (personal)

## Requirements

- Resolve cache addresses dynamically; never hardcode filenames.
- Preserve the exact original file size.
- Do not modify cache metadata.
- Support restoring original audio files.
- Rediscover entries after cache resets or Teams updates.

## Validation

- Ping replacement tested on New Teams Desktop.
- Incoming call replacement tested on a real call
