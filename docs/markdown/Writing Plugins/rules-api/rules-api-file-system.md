---
title: "File system"
slug: "rules-api-file-system"
excerpt: "How to safely interact with the file system in your plugin."
hidden: false
createdAt: "2020-07-01T04:40:26.783Z"
---
It is not safe to use functions like `open` or the non-pure operations of `pathlib.Path` like you normally might: this will break caching because they do not hook up to Pants's file watcher.

Instead, Pants has several mechanisms to work with the file system in a safe and concurrent way.

> 🚧 Missing certain file operations?
>
> If it would help you to have a certain file operation, please let us know by either opening a new [GitHub issue](https://github.com/pantsbuild/pants/issues) or by messaging us on [Slack](doc:the-pants-community) in the #plugins room.

Core abstractions: `Digest` and `Snapshot`
------------------------------------------

The core building block is a `Digest`, which is a lightweight reference to a set of files known about by the engine.

- The `Digest` is only a reference; the files are stored in the engine's persistent [content-addressable storage (CAS)](https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database).
- The files do not need to actually exist on disk.
- Every file uses a relative path. This allows the `Digest` to be passed around in different environments safely, such as running in a temporary directory locally or running through remote execution.
- The files may be binary files and/or text files.
- The `Digest` may refer to 0 - n files. If it's empty, the digest will be equal to `pants.engine.fs.EMPTY_DIGEST`.
- You will never create a `Digest` directly in rules, only in tests. Instead, you get a `Digest` by using `CreateDigest` or `PathGlobs`, or using the `output_digest` from a `Process` that you've run.

Most of Pants's operations with the file system either accept a `Digest` as input or return a `Digest`. For example, when running a `Process`, you may provide a `Digest` as input.

A `Snapshot` composes a `Digest` and adds the useful properties `files: tuple[str, ...]` and `dirs: tuple[str, ...]`, which store the sorted file names and directory names, respectively. For example:

```python
Snapshot(
    digest=Digest(
        fingerprint="21bcd9fcf01cc67e9547b7d931050c1c44d668e7c0eda3b5856aa74ad640098b",
        serialized_bytes_length=162,
    ),
    files=("f.txt", "grandparent/parent/c.txt"),
    dirs=("grandparent", "grandparent/parent"),
)
```

A `Snapshot` is useful when you want to know which files a `Digest` refers to. For example, when running a tool, you might set `argv=snapshot.files`, and then pass `snapshot.digest` to the `Process` so that it has access to those files.

Given a `Digest`, you may use the engine to enrich it into a `Snapshot`:

```python
from pants.engine.fs import Digest, Snapshot
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    snapshot = await Get(Snapshot, Digest, my_digest)
```

`CreateDigest`: create new files
--------------------------------

`CreateDigest` allows you to create a new digest with whichever files you would like, even if they do not exist on disk.

```python
from pants.engine.fs import CreateDigest, Digest, FileContent
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    digest = await Get(Digest, CreateDigest([FileContent("f1.txt", b"hello world")]))
```

The `CreateDigest` constructor expects an iterable including any of these types:

- `FileContent` objects, which represent a file to create. It takes a `path: str` parameter, `contents: bytes` parameter, and optional `is_executable: bool` parameter with a default of `False`.
- `Directory` objects, which can be used to create empty directories. It takes a single parameter: `path: str`. You do not need to use this when creating a file inside a certain directory; this is only to create empty directories.
- `FileEntry` objects, which are handles to existing files from `DigestEntries`. Do not manually create these.

This does _not_ write the `Digest` to the build root. Use `Workspace.write_digest()` for that.

`PathGlobs`: read from filesystem
---------------------------------

`PathGlobs` allows you to read from the local file system using globbing. That is, sets of filenames with wildcard characters.

```python
from pants.engine.fs import Digest, PathGlobs
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    digest = await Get(Digest, PathGlobs(["**/*.txt", "!ignore_me.txt"]))
```

- All globs must be relative paths, relative to the build root.
- `PathGlobs` uses the same syntax as the `sources` field, which is roughly Git's syntax. Use `*` for globs over just the current working directory, `**` for recursive globs over everything below (at any level the current working directory), and prefix with `!` for ignores.
- `PathGlobs` will ignore all values from the global option `pants_ignore`.

By default, the engine will no-op for any globs that are unmatched. If you want to instead warn or error, set `glob_match_error_behavior=GlobMatchErrorBehavior.warn` or `GlobMatchErrorBehavior.error`. This will require that you also set `description_of_origin`, which is a human-friendly description of where the `PathGlobs` is coming from so that the error message is helpful. For example:

```python
from pants.engine.fs import GlobMatchErrorBehavior, PathGlobs

PathGlobs(
    globs=[shellcheck.options.config],
    glob_match_error_behavior=GlobMatchErrorBehavior.error,
    description_of_origin="the option `--shellcheck-config`",
)
```

If you set `glob_match_error_behavior`, you may also want to set `conjunction`. By default, only one glob must match. If you set `conjunction=GlobExpansionConjunction.all_match`, then all globs must match or the engine will warn or error. For example, this would fail, even if the config file existed:

```python
from pants.engine.fs import GlobExpansionConjunction, GlobMatchErrorBehavior, PathGlobs

PathGlobs(
    globs=[shellcheck.options.config, "does_not_exist.txt"],
    glob_match_error_behavior=GlobMatchErrorBehavior.error,
    conjunction=GlobExpansionConjunction.all_match,
    description_of_origin="the option `--shellcheck-config`",
)
```

If you only need to resolve the file names—and don't actually need to use the file content—you can use `await Get(Paths, PathGlobs)` instead of `await Get(Digest, PathGlobs)` or `await Get(Snapshot, PathGlobs)`. This will avoid "digesting" the files to the LMDB Store cache as a performance optimization. `Paths` has two properties: `files: tuple[str, ...]` and `dirs: tuple[str, ...]`.

```python
from pants.engine.fs import Paths, PathGlobs
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    paths = await Get(Paths, PathGlobs(["**/*.txt", "!ignore_me.txt"]))
    logger.info(paths.files)
```

`DigestContents`: read contents of files
----------------------------------------

`DigestContents` allows you to get the file contents from a `Digest`.

```python
from pants.engine.fs import Digest, DigestContents
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    digest_contents = await Get(DigestContents, Digest, my_digest)
    for file_content in digest_contents:
        logger.info(file_content.path)
        logger.info(file_content.content)  # This will be `bytes`.
```

The result will be a sequence of `FileContent` objects, which each have a property `path: str` and a property `content: bytes`. You may want to call `content.decode()` to convert to `str`.

> 🚧 You may not need `DigestContents`
>
> Only use `DigestContents` if you need to read and operate on the content of files directly in your rule.
>
> - If you are running a `Process`, you only need to pass the `Digest` as input and that process will be able to read all the files in its environment. If you only need a list of files included in the digest, use `Get(Snapshot, Digest)`.
>
> - If you just need to manipulate the directory structure of a `Digest`, such as renaming files,  use `DigestEntries` with `CreateDigest` or use `AddPrefix` and `RemovePrefix`. These avoid reading the file content into memory.

> 🚧 Does not handle empty directories in a `Digest`
>
> `DigestContents` does not have a way to represent empty directories in a `Digest` since it is only a sequence of `FileContent` objects. That is, passing the `FileContent` objects to `CreateDigest` will not result in the original `Digest` if there were empty directories in that original `Digest`. Use `DigestEntries` instead if your rule needs to handle empty directories in a `Digest`.

`DigestEntries`: light-weight handles to files
----------------------------------------------

`DigestEntries` allows a rule to obtain the filenames (with content digests) and empty directories from a `Digest`. The value of a `DigestEntries` is a sequence of `FileEntry` and `Directory` objects representing files and empty directories in the `Digest`, respectively. That sequence can be passed to `CreateDigest` to recreate the original `Digest`.

This is useful if you need to manipulate the directory structure of a `Digest` without actually needing to bring the file contents into memory (which is what occurs if you were to use `DigestContents`).

```python
from pants.engine.fs import Digest, DigestEntries, Directory, FileEntry
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    digest_entries = await Get(DigestEntries, Digest, my_digest)
    for entry in digest_entries:
        if isinstance(entry, FileEntry):
            logger.info(entry.path)
            logger.info(entry.file_digest)  # This will be digest of the content.
        elif isinstance(entry, Directory):
            logger.info(f"Empty directory: {entry.path}")

```

`MergeDigests`: merge collections of files
------------------------------------------

Often, you will need to provide a single `Digest` somewhere in your plugin—such as the `input_digest` for a `Process`—but you may have multiple `Digest`s that you want to use. Use `MergeDigests` to combine them all into a single `Digest`.

```python
from pants.engine.fs import Digest, MergeDigests
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    digest = await Get(
        Digest,
        MergeDigests([downloaded_tool_digest, config_file_digest, source_files_snapshot.digest],
    )
```

- It is okay if multiple digests include the same file, so long as they have identical content.
- If any digests have different content for the same file, the engine will error. Unlike Git, the engine does not attempt to resolve merge conflicts.
- It is okay if some digests are empty, i.e. `EMPTY_DIGEST`.

`DigestSubset`: extract certain files from a `Digest`
-----------------------------------------------------

To get certain files out of a `Digest`, use `DigestSubset`.

```python
from pants.engine.fs import Digest, DigestSubset, PathGlobs
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    new_digest = await Get(
        Digest, DigestSubset(original_digest, PathGlobs(["file1.txt"])
    )
```

See the section `PathGlobs` for more details on how the type works.

`AddPrefix` and `RemovePrefix`
------------------------------

Use `AddPrefix` and `RemovePrefix` to change the paths of every file in the digest, while keeping the file contents the same.

```python
from pants.engine.fs import AddPrefix, Digest, RemovePrefix
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    added_prefix = await Get(Digest, AddPrefix(original_digest, "new_prefix/subdir"))
    removed_prefix = await Get(Digest, RemovePrefix(added_prefix, "new_prefix/subdir"))
    assert removed_prefix == original_digest
```

`RemovePrefix` will error if it encounters any files that do not have the requested prefix.

`Workspace.write_digest()`: save to disk
----------------------------------------

To write a digest to disk in the build root, request the type `Workspace`, then use its method `.write_digest()`.

```python
from pants.engine.fs import Workspace
from pants.engine.rules import goal_rule

@goal_rule
async def run_my_goal(..., workspace: Workspace) -> MyGoal:
    ...
    # Note that this is a normal method; we do not use `await Get`.
    workspace.write_digest(digest)
```

- The digest will always be written to the build root; you cannot write to arbitrary locations on your machine.
- You may set the optional parameter `path_prefix: str` with a relative path.

`Workspace` is a special type that can only be requested in `@goal_rule`s because it is only safe to write to disk in a `@goal_rule`. So, a common pattern is for "downstream" rules to return a `Digest` with the contents they want to write to disk, and then the `@goal_rule` aggregating all the results and writing them to disk. For example, for the `fmt` goal, each `FmtResult` includes a `digest` field.

For better performance, avoid calling `workspace.write_digest` multiple times, such as in a `for` loop. Instead, first, merge all the digests, then write them in a single call.

Bad:

```python
for digest in all_digests:
   workspace.write_digest(digest)
```

Good:

```python
merged_digest = await Get(Digest, MergeDigests(all_digests))
workspace.write_digest(merged_digest)
```

`DownloadFile`
--------------

`DownloadFile` allows you to download an asset using a `GET` request.

```python
from pants.engine.fs import DownloadFile, FileDigest
from pants.engine.rules import Get, rule

@rule
async def demo(...) -> Foo:
    ...
    url = "https://github.com/pantsbuild/pex/releases/download/v2.1.14/pex"
    file_digest = FileDigest(
        "12937da9ad5ad2c60564aa35cb4b3992ba3cc5ef7efedd44159332873da6fe46",
        2637138
    )
    downloaded = await Get(Digest, DownloadFile(url, file_digest)
```

`DownloadFile` expects a `url: str` parameter pointing to a stable URL for the asset, along with an `expected_digest: FileDigest` parameter. A `FileDigest` is like a normal `Digest`, but represents a single file, rather than a set of files/directories. To determine the `expected_digest`, manually download the file, then run `shasum -a 256` to compute the fingerprint and `wc -c` to compute the expected length of the downloaded file in bytes.

Often, you will want to download a pre-compiled binary for a tool. When doing this, use `ExternalTool` instead for help with extracting the binary from the download. See [Installing tools](doc:rules-api-installing-tools).

> 🚧 HTTP requests without digests are unsafe
>
> It is not safe to use `DownloadFile` for mutable HTTP requests, as it will never ping the server for updates once it is cached. It is also not safe to use the `requests` library or similar because it will not be cached safely.
>
> You can use a `Process` with uniquely identifying information in its arguments to run `/usr/bin/curl`.
