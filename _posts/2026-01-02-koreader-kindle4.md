---
id: 212
title: Install KOReader on Kindle 4
last_modified_at: 2026-01-02T21:00:00+01:00
permalink: /2026/01/02/koreader-kindle4
categories:
  - Configuration
---
KOReader is an eBook application optimized for E-Ink readers.
It can even be installed on old Kindle devices.
The manual explains to required steps.

<!--more-->

# Installation Instructions

Almost all required files can be downloaded from a [thread on mobileread.com](https://www.mobileread.com/forums/showthread.php?t=225030) ([Archive](https://web.archive.org/web/20260102124310/https://www.mobileread.com/forums/showthread.php?t=225030)).
The updated device certificates can be downloaded from another [post on mobileread.com](https://www.mobileread.com/forums/showpost.php?p=4506164&postcount=1295) ([Archive](https://web.archive.org/web/20260102125339/https://www.mobileread.com/forums/showpost.php?p=4506164&postcount=1295)).
KOReader can be downloaded from [Github](https://github.com/koreader/koreader/releases/tag/v2025.10).

The procedure is as follows:
* Jailbreak Kindle
* Install developer certificates to allow custom applications (such as KUAL)
* Install KUAL launcher
* Install KOReader

## Jailbreak Kindle
* Download `kindle-k4-jailbreak-1.8.N-r18977.tar.xz` and extract it
* Follow README
  * Copy `data.tar.gz`, `ENABLE_DIAGS`, `diagnostic_logs` to the root of Kindle and eject it from PC
  * Restart (`Menu -> Settings -> Menu -> Restart`)
  * After restart, select `D) Exit, Reboot or Disable Diags`
  * Select `R) Reboot System" and "Q) To continue`

## Add Developer Certificates
* Download `kindle-mkk-20141129-r18833.tar.xz` and extract it
* Place file `Update_mkk-20141129-k4-ALL_install.bin` in root of Kindle and eject it from PC
* Install via `Menu -> Settings -> Menu -> Update Your Kindle`

## Add Updated Developer Certificates
* Download `DevCerts-20250419-KeyStore.zip` and extract it
* Place file `Update_mkk-20250419-k4-ALL_keystore-install.bin` in root of Kindle and eject it from PC
* Install via `Menu -> Settings -> Menu -> Update Your Kindle`

## Add KUAL
* Download `KUAL-v2.7.37-gfcb45b5-20250419.tar.xz` and extract it
* Place file `KUAL-KDK-1.0.azw2` in `documents` folder of Kindle

## Add KOReader
* Download `koreader-kindle-v2025.10.zip` and extract it
* Place folders into root of Kindle and eject it from PC


# Usage

* Launch KUAL (should be part of the usual books menu)
* Launch KOReader (e.g., without framework to save resources)


# Further Hints on Usage of KOReader
* Integration of reading state in Booklore in [manual](https://booklore-app.github.io/booklore-docs/docs/integration/koreader) ([Archive](https://web.archive.org/web/20260102131142/https://booklore-app.github.io/booklore-docs/docs/integration/koreader))
* Integration of database from Booklore in [manual](https://booklore-app.github.io/booklore-docs/docs/integration/opds) ([Archive](https://web.archive.org/web/20260102131154/https://booklore-app.github.io/booklore-docs/docs/integration/opds))
