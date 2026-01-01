---
id: 211
title: Download of multiple audio tracks with yt-dlp
last_modified_at: 2026-01-01T21:00:00+01:00
permalink: /2026/01/01/ytdlp-multilang
categories:
  - Uncategorized
---
`yt-dlp` can download videos from many websites.
If the website offers multiple audio tracks for a video, here is how to download and integrate it into the video file.

<!--more-->

The parameters for the script are
1. The URL
2. The expected file name

```sh
#!/bin/bash

set -eu -o pipefail

echo "Downloading ${1} as ${2}"

AUDIO=$(yt-dlp --dump-json "${1}" | jq -r '[[.formats[] | select(.vcodec == "none") | .language] | sort | unique[] | "+bestaudio[language=\(.)]"] | join("")')

yt-dlp --write-subs --sub-langs 'deu,eng,fra' --audio-multistreams --format "bestvideo${AUDIO}" --embed-metadata --merge-output-format 'mp4' --write-thumbnail --output "${2}.%(ext)s" "${1}"
```