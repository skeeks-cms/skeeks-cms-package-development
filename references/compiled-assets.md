# Compiled css/js files (assets-auto-compress)

`skeeks/yii2-assets-auto-compress` merges the css and js files of a page into
`assets/css-compress` and `assets/js-compress`; `skeeks/cms-assets-auto-compress`
applies its settings in CMS and owns the CMS integration (job, schedule, console).
Verified on the Windows stand and against production measurements, 2026-10-26
(yii2-assets-auto-compress 1.4.6.0, cms-assets-auto-compress 1.6.4.0, unreleased).

## Names follow content, not sources

Source URLs carry the asset directory hash (`crc32(path . filemtime(dir) . …)`,
Yii `AssetManager::hash`) and `?v=filemtime` (`appendTimestamp`). A name derived
from those URLs changes on every `touch` of a source directory and every package
reinstall, even when nothing changed. On skeeks.com this produced 300+300 files a
day, 54 % (css) and 64 % (js) of them byte-identical copies, never removed.

So a compiled file is named `md5(content)`, and the set of sources maps to it
through `.index/<md5(source URLs . content settings)>`, a file holding the name.
Republishing adds an index entry, not a copy. Write both through `.tmp-*` plus
`rename`; a plain `fopen("w")` lets a parallel request read a partial file.
`getSettingsHash()` covers only options that change content plus a scheme
version; serializing the whole component tied names to unrelated state.

## Page cost

The hit path runs on every page: one small index read and two stats, about the
same as the former `file_exists` + `filemtime` (measured 5.4 vs 5.1 µs per file).
A miss adds `md5` of the content (about 2 ms for 1.5 MB), negligible next to
minification. Do not read compiled content or write on every hit.

## Usage marks and cleanup

atime and ctime cannot tell whether a page still uses a file: backups read every
file (one atime for 1 364 files) and a deploy's `chown -R` refreshes ctime. The
component therefore refreshes the mtime of the index entry and of the compiled
file at most once per `usageTouchInterval` (1 hour) on a hit; `?v=` is gone, so
that mtime is free for marking.

`CompiledFilesCleaner` deletes, with a dry-run mode and only stat()/listing:
index entries unused for `days` (+ the touch interval); compiled files that no
live entry references and are older than `days`; `.tmp-*` older than an hour.
Before each unlink it re-stats the file, so a page that marked it meanwhile wins.
Files of the previous scheme have no entry; they go only when `days` have also
passed since `.index/.since` (written on the first new-scheme compile), because
HTML rendered before the upgrade still points to them. No `.since`, no deletion.

A deleted compiled file is rebuilt by the next page that renders it; a direct
request from older HTML gets a 404 from the web server. There is no server page
cache on SkeekS sites, so retention since last use (default 7 days) is the guard.

## CMS integration

`cms-assets-auto-compress/src/config/common.php` (web and console) registers,
guarded by `class_exists`, the native type `assets-auto-compress.cleanup` on
`maintenance` (skip overlap, idempotent, installation-wide resource key, admin
permission) and the daily `cmsAgent.jobs` schedule. Payload
`{"days": 7, "dryRun": false}`. Directories come from
`cms->tmpFolderScheme['assets']`, the list `cms/cache/flush-assets` clears. The
console route is `assets-auto-compress/cleanup [--dry-run] [--days=N]`.

On a local stand whose merge plan is built from the installed lock, a changed
`extra.config-plugin` of a symlinked package is invisible until the package is
installed by version; `cmsAgent/init` there also applies every other pending
schedule of the stand, so check and disable foreign ones afterwards.
