---
id: 161
title: Collection of Useful Command Line Snippets
last_modified_at: 2018-09-02T21:04:21+01:00
permalink: /2018/09/02/collection-of-useful-command-line-snippets/
categories:
  - Programming
---
Using command line utilities is often convenient because good UI tools often do not exist. Anyway, the downside of using such tools is the high amount of parameters. In this post, I just collect a bunch of useful snippets that I often use but cannot remember.

<!--more-->

## Bash

Copy the modification time stamp from files.  
`touch -r original destination`

Rename all files to their modification date  
`for f in *; do mv -- "$f" "$(date -d @$(stat -c %Y "$f") +%Y-%m-%d_%H:%M:%S).$(echo $f | awk -F . '{print $NF}' | tr '[:upper:]' '[:lower:]')" ; done`

## FFMpeg

Rotate a video (0 = 90CounterCLockwise and Vertical Flip, 1 = 90Clockwise, 2 = 90CounterClockwise, 3 = 90Clockwise and Vertical Flip) and cut it from start `ss` for a duration of `t`:  
`ffmpeg -i input.mp4 -vf "transpose=2" -ss 00:00:01 -t 00:00:33 output.mp4`