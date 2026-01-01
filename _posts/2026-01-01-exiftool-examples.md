---
id: 210
title: Examples for exiftool
last_modified_at: 2026-01-01T21:00:00+01:00
permalink: /2026/01/01/exiftool-examples
categories:
  - Uncategorized
---
`exiftool` is a powerful tool for working with EXIF data but it is often not intuitive.
Therefore, I collect a few useful examples on how to use it here.

<!--more-->

## mtime from create date (exif)
```sh
exiftool '-FileModifyDate<DateTimeOriginal' pic.jpg
```

## set create date (exif) from mtime
```sh
exiftool '-DateTimeOriginal<FileModifyDate' pic.jpg
```

## offset the create date (exif)
The command adds (first line) or subtracts (second line) a given offset.
The format is `years:months:days hours:minutes:seconds`.
```sh
exiftool -DateTimeOriginal+="1:2:3 4:5:6" pic.jpg
exiftool -DateTimeOriginal-="1:2:3 4:5:6" pic.jpg
```

## set create date (exif) to fixed value
```sh
exiftool -time:all="2016:01:20 13:30:45" *.mp4
```