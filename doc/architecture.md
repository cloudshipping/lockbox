# Requirements

* backup of arbitrary file system data into a remote place,
* disaster recovery backup only, that is, restore operations are only
  there for recovering entire file systems, not restoring accidentally
  deleted single files,
* snapshot backup, that is, backup only happens "every now and then",
* the remote place only supports storing immutable, named files,
* limited trust into the remote place.


# Concept

* Backup happens into a sequence of archives each containing the changes
  from the last archive only.
* Each archive is an encrypted tar archive containing a number of objects
  stored as the files of the archive.
* Each archive has a unique name, each object is identified by a SHA-1 of its
  content.
* The file names of the files in each archive are such that it appears as
  if each file in the archive is stored in a directory with the name of
  the archive.
* Most objects are just binary blobs containing the unadulterated content
  of a file in the file system backed up.
* The only special type of object are directory listings. They contain a
  list of directory entries including a name, some meta data and the file
  name of the object representing the entry.
* Each archive has, as its very first object, a directory listing linking
  to the directory listings for the topmost directories backed up. The
  name of this file is a number in hexadecimal ASCII representation
  growing from archive to archive. (This could be a time stamp but it
  seems safer to not connect any meaning to it.)
* An extension for later are diff objects which don’t contain the
  full content of a file but the difference to an earlier version in some
  form.


# Backup Daemon

With the above, the backup daemon needs to keep a list of all the hashes
of the objects stored in earlier archives as well as which archive they
are in. When backing up, the daemon needs to traverse the directories
depth-first and calculate the hash for each object it finds. If the hash
is already known, all is good. If there is a previously unknown hash,
the object’s current content needs to be added to the new archive. In
both cases, the daemon marks the hash as currently in use. When the
traversal is complete, the daemon finishes up the archive and uploads it.
It then removes the unused hashes from its state. If by doing this it
discovers that all hashes from a certain archive are not used anymore,
it can delete the archive from the remote store.


# Restore

A simple restore procedure would download all archives and untar them
into a common local directory. From those files in the top directory, it
would read the one with the largest number as its name to discover the
hashes for the top directories. It can then start moving through directories
and recreate the structure.

