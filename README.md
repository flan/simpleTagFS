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
- tags will support spaces and other POSIX-filename-friendly characters, just not leading `-`s because they're reserved for negation

When the VFS component is mounted, its reference directory is recursively scanned for ".tags" files, which are used to populate its own data-map. For efficiency, there will probably be a ".tags.sqlite3" file placed in each scanned directory that maintains the compiled status of all files within, with mtime/size notes on which files have been processed to reduce IO load.
- if a parent directory's .tags file has changed, determined by comparing the list of parent tags to those from an always-performed scan, all child databases need to be regenerated
- pull the dentry and run it as a set against the records in the database: if a file is new or has changed, upsert the tag-records, which will ideally each be separate tables that just contain filenames, but could be indexed tuples, or maybe a tuple of (tag, JSON array of filenames); some experimentation and thought will be required to determine what's most efficient
- if a .tags file corresponding to a previous record is missing, drop it from the database
- each of these sqlite files is kept open through the VFS's lifetime and used to merge data to satisfy queries

There will be two modes of operation, configurable on a per-instance basis: responsive and cached
- in responsive mode, every watched directory is tracked by inotify and the SQLite databases may be updated in response to changes; adding and removing directories will be supported
- in cached mode, only the data used on-mount is served, though SIGHUP can be used to trigger a reload

Within the mounted VFS, tagged files are exposed as symlinks in a flat directory. In general, they will retain their original names, but in the event of a collision, the file's inode ID will be inserted before the file-extension, if one is present, like "hello.1234567890.ogg", ensuring reasonable deterministic consistency across mounts with reasonable sortability, so long as the underlying data remains unchanged and doesn't move between filesystems.

Within these directories, all tags not yet present in the chain will be enumerated as subdirectories, including their negated forms, so AND/NAND groupings may be easily navigated.

Deleting a file from one of these directories will remove (or set, in the case of negation) the tag associated with the immediate parent, so someone could naively apply a slew of tags to a library and work backwards to the subset that they actually want; this change takes immediate effect in both responsive mode and cached mode (by explicitly triggering a rebuild of the affected directories as though the inotify signal had been received) and requires another mount option to be specified to allow manipulation.

A number of scripts will also be needed:
- enumerate tags (including counts, to find typos)
- replace tags
- some tools that do more complex tag-grouping, like OR and value-based range-handling
- bulk tag-apply/remove over files (directory-level ".tags" files must be handled manually)
- queries to see the tags applied to a specific file
- enumeration of bare files, optionally creating empty ".tags" for them
