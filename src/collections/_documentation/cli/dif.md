---
title: 'Debug Information Files'
sidebar_order: 3
---

Debug information files allow Sentry to extract stack traces and provide more
information about crash reports for most compiled platforms. `sentry-cli` can be
used to validate and upload debug information files. For more general
information, refer to [_Debug Information Files_]({%- link
_documentation/workflow/debug-files.md -%}).

{% capture __alert_content -%}
Source maps, while also being debug information files, are handled differently
in Sentry. For more information see [Source Maps in sentry-cli]({%- link
_documentation/cli/releases.md -%}#sentry-cli-sourcemaps).
{%- endcapture -%}
{%- include components/alert.html
  title="Note"
  content=__alert_content
%}

## Checking Files

Not all debug information files can be used by Sentry. To see if they are usable
or not, you can use the `sentry-cli difutil check` command:

```bash
$ sentry-cli difutil check mylibrary.so.debug

Debug Info File Check
  Type: elf debug companion
  Contained debug identifiers:
    > 924e148f-3bb7-06a0-74c1-36f42f08b40e (x86_64)
  Contained debug information:
    > symtab, debug
  Usable: yes
```

This will report the debug identifiers of the debug information file as well as
if it passes basic requirements for Sentry.

## Finding Files

If you see in Sentry's UI that debug information files are missing, but you are
not sure how to locate them, you can use the `sentry-cli difutil find` command
to look for them:

```bash
$ sentry-cli difutil find <identifier>
```

Additionally, `sentry-cli upload-dif` can automatically search for files in a
folder or ZIP archive.

## Creating Source Bundles

To get inline source context in stack traces in the Sentry UI, `sentry-cli` can
scan debug files for references to source code files, resolve them in the local
file system and bundle them up. The resulting source bundle is an archive
containing all source files referenced by a specific debug information file. 

This is particularly useful when building and uploading debug information files
are detached. In this case, a source bundle can be created when building and can
be uploaded at any later point in time with `sentry-cli upload-dif`.

To create a source bundle, use the `difutil bundle-sources` command on a list of
debug information files: 

```bash
$ sentry-cli difutil bundle-sources /path/to/files...
```

To create multiple source bundles for all debug information files, use the
command on each file individually. 

Alternatively, specify the `--include-sources` parameter to the upload command,
which generates source bundles on the fly during the upload. This requires that
the upload is performed on the same machine as the application build.

## Uploading Files

Use the `sentry-cli upload-dif` command to upload debug information files to
Sentry. The command will recursively scan the provided folders or ZIP archives.
Files that have already been upload are skipped automatically.

We recommend uploading debug information files when publishing or releasing your
application. Alternatively, files can be uploaded during the build process. See
[_Debug Information Files_]({%- link _documentation/workflow/debug-files.md -%})
for more information.

{% capture __alert_content -%}
You need to specify the organization and project you are working with because
debug information files work on projects. For more information about this refer
to [Working with Projects]({%- link _documentation/cli/configuration.md
-%}#sentry-cli-working-with-projects).
{%- endcapture -%}
{%- include components/alert.html
  title="Note"
  content=__alert_content
%}

A basic debug file upload can be started with:

```bash
$ sentry-cli upload-dif -o <org> -p <project> /path/to/files

> Found 2 debug information files
> Prepared debug information files for upload
> Uploaded 2 missing debug information files
> File processing complete:

     OK 1ddb3423-950a-3646-b17b-d4360e6acfc9 (MyApp; x86_64 executable)
     OK 1ddb3423-950a-3646-b17b-d4360e6acfc9 (MyApp; x86_64 debug companion)
```

### Upload Options

There are a few options you can supply to the upload command:

`--no-unwind`

: Do not scan for stack unwinding information. Specify this flag for builds with
  disabled FPO, or when stack walking occurs on the device. This usually
  excludes executables and libraries. They might still be uploaded, if they
  contain debug information.

`--no-debug`

: Do not scan for debug information. This will usually exclude debug companion
  files. They might still be uploaded, if they contain stack unwinding
  information.

`--include-sources`

: Scans the debug files for references to source code files. If referenced files
  are available on the local file system, they are bundled and uploaded as a
  source archive. This allows Sentry to resolve source context. Only specify
  this command when uploading from the same machine as the build. Otherwise, use
  `difutil bundle-sources` to generate the bundle ahead of time.

`--derived-data`

: Search for dSYMs in the derived data folder. Xcode stores its build output in
  this default location.

`--no-zips`

: By default, `sentry-cli` will open and search ZIP archives for debug files.
  This is especially useful when downloading builds from iTunes Connect or prior
  build stages in a CI environment. Use this switch to disable if your search
  paths contain large ZIP archives without debug files to speed up the search.

`--force-foreground`

: This option forces the upload to happen in the foreground. This only affects
  uploads invoked from Xcode build steps. By default, the upload process will
  detach when started from Xcode and finish in the background. If you need to
  debug the upload process it might be useful to force the upload to run in the
  foreground.

`--info-plist`

: Overrides the search path for `Info.plist`, useful if it is located in a
  non-standard location.

`--no-reprocessing`

: This parameter prevents Sentry from triggering reprocessing right away. It can
  be useful under rare circumstances where you want to upload files in multiple
  batches and you want to ensure that Sentry does not start reprocessing before
  some optional dSYMs are uploaded. Note though that someone can still in the
  meantime trigger reprocessing from the UI.

`--symbol-maps`

: Resolve hidden symbols in iTunes Connect builds using BCSymbolMaps. This is
  needed to symbolicate crashes if symbols were not uploaded to Apple when
  publishing the app in the AppStore.

### Symbol Maps

If you are hiding debug symbols from Apple, the debug files will not contain
many useful symbols. In that case, the sentry-cli upload will warn you that it
needs BCSymbolMaps:

```bash
$ sentry-cli upload-dif ...
> Found 34 debug information files
> Warning: Found 10 symbol files with hidden symbols (need BCSymbolMaps)
```

In this case, you need the BCSymbolMaps that match your files. Typically, these
are generated by the Xcode build process. Supply the `--symbol-maps` parameter
and point it to the folder containing the symbol maps:

```bash
$ sentry-cli upload-dif --symbol-maps path/to/symbolmaps path/to/debug/symbols
```

### Breakpad Files

In contrast to native debug files, Breakpad symbols discard a lot of information
that is not required to process minidumps. Most notably, inline functions are
not declared, such that Sentry is not able to display inline frames in stack
traces.

If possible, upload native debug files such as dSYMs, PDBs or ELF files instead
of Breakpad symbols.

## ProGuard Mapping Upload

`sentry-cli` can be used to upload ProGuard files to Sentry; however, in most
situations, you would use the [Gradle
plugin](https://github.com/getsentry/sentry-java) to do that. There are some
situations, however, where you would upload ProGuard files manually (for instance
when you only release some of the builds you are creating).

{% capture __alert_content -%}
You need to specify the organization and project you are working with because
ProGuard files work on projects. For more information about this refer to
[Working with Projects]({%- link _documentation/cli/configuration.md
-%}#sentry-cli-working-with-projects).
{%- endcapture -%}
{%- include components/alert.html
  title="Note"
  content=__alert_content
%}

The `upload-proguard` command is the one to use for uploading ProGuard files. It
takes the path to one or more ProGuard mapping files and will upload them to
Sentry. If you want to associate them with an Android app you should also point
it to a processed _AndroidManifest.xml_ from a build intermediate folder.
Example:

```bash
sentry-cli upload-proguard \
    --android-manifest app/build/intermediates/manifests/full/release/AndroidManifest.xml \
    app/build/outputs/mapping/release/mapping.txt
```

Since the sentry-java SDK needs to know the UUID of the mapping file, you need
to embed it in a `sentry-debug-meta.properties` file. If you supply
`--write-properties` that is done automatically:

```bash
sentry-cli upload-proguard \
    --android-manifest app/build/intermediates/manifests/full/release/AndroidManifest.xml \
    --write-properties app/build/intermediates/assets/release/sentry-debug-meta.properties \
    app/build/outputs/mapping/release/mapping.txt
```

### Upload Options

`--no-reprocessing`

: This parameter prevents Sentry from triggering reprocessing right away. It can
  be useful under rare circumstances where you want to upload files in multiple
  batches and you want to ensure that Sentry does not start reprocessing before
  some optional dSYMs are uploaded. Note though that someone can still in the
  meantime trigger reprocessing from the UI.

`--no-upload`

: Disables the actual upload. This runs all steps for the processing but does
  not trigger the upload (this also automatically disables reprocessing. This is
  useful if you just want to verify the mapping files and write the ProGuard
  UUIDs into a properties file.

`--require-one`

: Requires at least one file to upload or the command will error.
