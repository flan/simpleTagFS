# simpleTagFS
A simple FUSE-powered tagging system with an emphasis on metadata portability

This project was started because every other tagging system looked too complex for my needs. What I want is a tagging system with the following qualities:
- freedom to move files between directories, filesystems, and computers
- compatibility with traditional directory-based grouping, without an import/export step or constraints on structuring
- ability to manage tags through a remote filesystem share
- bulk tag-management as an option
- script-friendly processing for conversion to other systems if I ever find this was *too* simple
- support for multiple tagged directory instances, each serving data independently

While there will be a solid architecture in place to minimise inefficiencies, performance itself is not a project goal.


Pre-production notes:

This project will be implemented in Go, with additional management scripting handled in Python

Every file to be indexed will be handled by adding a ".tags" file alongside it, so "hello.ogg" will be tagged by "hello.ogg.tags". Directories can be tagged in the same way, with the ".tags" file applying to all files within, recursively. The contents of these files are linebreak-delimited and support comments.

Tagging syntax will be pretty simple:
- `hello` applies a tag called "hello"
- `-hello` removes any existing tag called "hello"
- `year=2000` applies a tag called "year" with a value of "2000"; a subsequent `year` tag replaces the previous value and unvalued tags are really like `hello=`
- `-year=2000` removes any tag called "year" that specifically has a value of "2000", while `-year` would remove the tag regardless of value
- tags will support spaces and other POSIX-filename-friendly characters, just not leading `-`s because they're reserved for negation (though this could turn into a trailing `-`, so that the AND and NAND subdirectories show up next to each other in file-managers)

When the VFS component is mounted, its reference directory is recursively scanned for ".tags" files, which are used to populate its own data-map. For efficiency, every directory is populated with a ".simpleTagFS" manifest file that maintains the status of all files within, with mtime/size notes on which files have been processed to reduce IO load and a list of relevant tags, *not* including those inherited from directories, null-character-delimited, one file per line so buffered iterators can parse them efficiently.
- for each mountpoint, an in-memory data-structure is prepared, which serves as an index of all tagged files within; its contents follow:
  - a map of relative directory paths, such as "ogg", "ogg/vorbis", "ogg/opus", each of which point to another structure:
    - a map of tag-strings to sets of (filename, inode ID) tuples, informed by tags inherited from directories
  - for effiiciency, the process of preparing the data-structure and building the .simpleTagFS manifests will likely be the same workflow: iterate over the existing file, iterate over the dentry, and if anything has changed, serialise the new data-structure into the directory
- to satisfy a query, every matching set in every directory is combined, then the sets are run against each other by boolean logic to produce the final list, used to assemble the synthetic dentry
- this structure should be pretty fast to populate, since, in the common case, each subdirectory only incurs two read operations: the dentry and the ".simpleTagFS" file; in the event of a change within, an additional read is only needed for each changed file, assuming it doesn't just inherit from the directory tree

There will be two modes of operation, configurable on a per-instance basis: responsive and cached
- in responsive mode, every subdirectory is tracked by inotify and .simpleTagFS files are recereated for each change, then reloaded into the mount's data-structure; adding and removing directories will be supported
- in cached mode, only the data used on-mount is served, though SIGHUP can be used to trigger a reload

Within the mounted VFS, tagged files are exposed as symlinks in a flat directory. In general, they will retain their original names, but in the event of a collision, the file's inode ID will be inserted before the file-extension, if one is present, like "hello.1234567890.ogg", ensuring reasonable deterministic consistency across mounts with reasonable sortability, so long as the underlying data remains unchanged and doesn't move between filesystems.

Within these directories, all tags not yet present in the chain will be enumerated as subdirectories, including their negated forms, so AND/NAND groupings may be easily navigated.

Deleting a file from one of these directories will remove (or set, in the case of negation) the tag associated with the immediate parent, so someone could naively apply a slew of tags to a library, maybe by creating some broad directory trees with per-level composite tagging, and work backwards to the subset that they actually want; this change takes immediate effect in both responsive mode and cached mode (by explicitly triggering a rebuild of the affected directories' .simpleTagFS files and replacing the relevant map-entry as though the inotify signal had been received) and requires another mount option to be specified to allow manipulation.

A number of scripts will also be needed:
- enumerate tags (including counts, to find typos)
- replace tags
- some tools that do more complex tag-grouping, like OR and value-based range-handling
- bulk tag-apply/remove over files (directory-level ".tags" files must be handled manually)
- queries to see the tags applied to a specific file
- enumeration of bare files, optionally creating empty ".tags" for them
- enumeration of abandoned ".tags" files, optionally allowing for removal
