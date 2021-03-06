# Kodi database management (MyVideos119.db)

Kodi is a great media player. It is even great at finding metadata around movies and the like (called scraping) and storing/displaying this data and images in the user interface. What Kodi doesn't do very well (read: doesn't do at all), is keeping track of your media files if you hapen to move these around (and don't we all do that, keeping the library organised?). In Kodi, there's only two things you can do: Clean the Library, which will *delete* all entries that are no longer in the expected location on the disk, even if you just moved or renamed the original media file, and create *new* ones in the location where the items reappeared, losing the view state (playcount and bookmark), or Rescrape (which will re-download all metadata from one of the online sources).

There are many 'howtos' around on managing file and directory paths in a MyVideos119.db. None of these (including the https://kodi.wiki/view/HOW-TO:Update_SQL_databases_when_files_move or https://kodi.wiki/view/HOW-TO:Update_Paths_In_MySQL) work(ed) very well for various reasons. Catching all variations and multiple occurrences of file paths, different kodi path notation, URL-encoded- and XML-escaped- (if you use the XML export) and UTF-8 strings (like Antonín Dvořák), then at the same time make sure the database remains consistent is not trivial and can't be caught in a simple 'search-replace' or a couple of SQL queries - that's why I started writing this tool, in need for some granular control over the process and being able to see what happend, what I screwed up and, if necessary, retrace my steps in the next run.

## kodidb_check

`kodidb_check` is a utility to check, clean up and manage the kodi media player database (MyVideos119.db)

This script provides a number of options to check, fix and clean up a Kodi MyVideos database. It can, among others:
- remap a path or a file, and make sure all referenced files are updated properly and related records are dragged along accordingly
- rename a file, path, or part of a path and make sure the database is updated properly
- do a consistency check on the directory structure as registered in the database, and fix it
- list duplicate files found in the database, and optionally provide a prompt to fix the ambiguity
- list files from the database that can no longer be found (located) on the local file system, and optionally provide a prompt to fix the missing items
- save all database-modifying operations in a log file (`history.log`), so you can review them or keep them for record-keeping. The commands are printed in a bash run-ready form so you can even extract them for replay or save them in a script (you need to run `cut -f 2 history.log` to strip the dates).

Definitions:
  - "kodi" is referred to as the machine you run Kodi on. This can be a linux computer, an embedded linux player (OpenELEC, CoreELEC etc.) or MacOSX.
  - "local" is the machine you run these scripts on. This can be a linux computer or MacOSX. Theoretically, it could also be the embedded linux player, but I found the bash there not capable of using arrays. I haven't tested any of this with Windows.
- The script works off the concept of (media) files. The files have name and a path (directory) where they are stored. Files can be '*in use*' and therefore important to you and for the script to manage. *In use* means:
  - The file is (i.e. is used by) a kodi movie (NOTE: the script doesn't do TV Shows or Music Videos)
  - The file was played (has a play count > 0)
  - The file playback was in progress (this is called a 'bookmark' and allows you to resume playback of the file where you left off)
- Files that *are* in use need to be treated carefully - the watched state, for instance, particularly for files that are not movies, is an important indicator in the file browser interface in Kodi. You can see what you've watched through the tick mark (or more importantly, what you've not watched yet, or which files are 'in progress' of watching through the halfopen circle mark: the bookmark is important to allow you to resume viewing where you left off last time.
- Files that are *not* in use don't need to be kept in the database. Why would you? Over the last 7 years that I've used Kodi, the database contracted around 35k files (and paths) that were just traversed or scanned at some time, then moved, removed, disappeared, but never played (in full or partial) or became used by a movie - these can all be cleaned out - no harm done. Finding and fixing the stuff that needs to be kept is the trick here. The script got me from:

```
$ kodidb_check --list > dblist.lst
$ kodidb_check --summary < dblist.lst
f_founddup______: 97
f_foundlike_____: 10587
f_foundone______: 2769
f_hasbookmark___: 1289
f_hasmovie______: 1031
f_hasplaycount__: 3317
f_unref_________: 34945
f_notfound______: 13229
p_content_______: 1
p_found_________: 313
p_noparent______: 595
p_notfound______: 540
p_root__________: 1
p_unref_________: 1
```
to
```
f_hasbookmark___: 867
f_hasmovie______: 1031
f_hasplaycount__: 2331
p_content_______: 1
p_found_________: 563
p_noparent______: 5
p_notfound______: 5
p_root__________: 1
p_unref_________: 5
```

IMPORTANT:
- First, set up the `sources` array. The script works off the concept that all media files are tied to one of the 'kodiroot' directories set up in the script: they should point to the highest order directory that the kodi box shares
with the local machine. The idea is that Kodi has a certain view on the file system that holds the media, and the local machine has the same view, but a different root path. `sources` bring these together through a (root) path mapping, and, if necessary, also takes protocol into account. Kodi, for instance, can use paths that start with `smb://x.x.x.x/` and the local machine can have the media directory mounted on `/mnt/media/`, or `/Volumes/media` if you own a Mac. This will be different for every setup, hence the mapping. If you examine the `kodidb_check -l` output, you'll quickly see which paths kodi uses to access your media files. Some examples are given in the script.
- Secondly, set up the `_createlocalindex()` function - this is where the list of local media files is constructed. The by default, the function will scan the directory trees referenced by the `sources` array (the localpath part), and treat them as the master source to crossreference against the database, so the scanning can take a little while. The result is stored in `fileindex.fst`, and only scanned once (delete this file if the `sources` change or the filesystem contents have changed, and it will be re-scanned).
- The script does not touch (write/modify) your filesystem. It does read the file system to find files and check if (media) files and directories are there. This is to make sure the database, once put back on the kodi box, still works. If you're paranoid, you can mount the file system r/o and see that the script still works. Some command options don't even access the filesystem or `fileindex.fst` at all and run purely on the database.
- If you have multiple sources, like several removable disks with media or various shares with media, just add them the `sources` array by mapping the kodi view to the local view.
- The script *does* modify the MyVideos119.db database. Always make a (or several) backup copies before attempting a cleanup. Hint: call the backup copy something different, like MyVideos119.db.org, so it can not accidentally be used by `kodidb_check`. Use the `kodi_check_replay` script to explicitly store and work on separate phases of the cleanup, with multiple backup points to retrace your steps.

Constraints:
- Movies are not touched, regardless whether they are watched or not (these can be cleaned up with 'Clean Library' in kodi or 'texturecache.py vclean' on the kodi machine)
- Does not scan or check for Music Videos, Episodes, TV Shows, Pictures, Music, sorry... That's because I don't use these types of media. If you use these, they do not classify the scanned files as '*in use*' and the files may be cleaned up. If you despereately need this feature, drop me a line.
- The script works off the MyVideos119.db database, not `.NFO` files that may be stored alongside the media.

Limitations:
- The 'kodi' system: tested on version 119 of MyVideos.db, from a Kodi 19 'Matrix' running on an embedded linux player (so it has / in the kodi paths, not \\). I have no clue what a database looks like when Kodi is run as an application on a Windows computer.
- The 'local' system: tested on linux-gnu and darwin21 

Dependencies:
- The script uses sqlite3, which is the format of the Kodi databases.
- The script relies on bash >=4 (for the mapfile builtin function). MacOSX 'local' machines may need to install a newer bash and/or sqlite3 from MacPorts or Brew (PS> when you do, the shebang (#!/bin/bash at the beginning) typically points to the wrong (v3) bash, so you need to adjust there).
- The script uses mac2syn (https://github.com/hwdbk/synology-scripts/blob/master/mac-nfd-conversion/mac2syn) for converting MacOSX NFD (normalization form decomposed) UTF-8 strings to 'normal' UTF-8. The invocation is 'iffed' around an OSTYPE check, so if you're not running MacOSX as a local machine, it should not be a problem.

```
usage: kodidb_check [options] [args]
        [-db, --database file]                   use file as database instead of the default MyVideos119.db, followed by one of the functions below:
         -l,  --list                             will scan the data base and lists output that can be filtered with grep. idFile or idPath is in the 2nd column. does not modify the database
         -c,  --check                            will scan the data base and lists inconsistencies. does not modify the database
         -df, --delete-file [-f] [-g] file ...   delete the file(s) specified by localpath or idFile and associated record(s) from the database, if allowed (i.e. not in use by a movie), or force [-f]
         -dp, --delete-path [-f] [-g] path ...   cleanup path record(s) specified by localpath or idPath from the database incl. subpaths, insofar allowed (i.e. not a scrape dir and no subpaths/files), or force [-f]
         -fp, --fix-paths                        scan and fix the directory structure in the database
         -rf, --remap-file file newlocalpath     remap the file specified by localpath or idFile to the (existing) newlocalpath file
         -rp, --remap-path path newlocalpath     remap the path specified by localpath or idPath to the (existing) newlocalpath directory
         -mw, --mark-watched[=<num>] file ...    bump the playcount of the file(s) specified by localpath or idFile or set playcount to <num>, if specified. creates the file(s) in the database if not exists

    utilities for analysis and cleanup of the database (all commands read an output log formatted/produced by 'kodidb_check -l' from stdin):
              --summary                          print summary of the various categories of files in log
              --delete-unref                     remove all read 'unref' file records from database (i.e. files that are labelled as 'unreferenced' in the log)
              --clean-roots                      cleanup the directory structure starting from the root dir(s) in the database, insofar paths are not used
              --remap-foundone                   remap all read 'foundone' files to the exact match listed in the log
              --remap-foundlike                  remap all read 'foundlike' files to the almost exact match listed in the log (matched without extension)
              --list-founddup [--pick]           list the read duplicate matches in a more readable form and, with --pick, present a prompt where the user can pick any of the found duplicates
              --list-notfound [--pick]           list all read 'notfound' files and, with --pick, present a prompt remap the file interactively. use this if you can drop file paths on the prompt
              --rename-founddup oldstr newstr    remap files with duplicate matches after substituting oldstr with newstr in the full path
              --rename-notfound oldstr newstr    remap files which weren't found after substituting oldstr with newstr in the full path
```
Note: for the 2nd set of options (starting with `--summary`) it is important to know that these run off the result of a `kodidb_check -l`. It then helps to store this result (a database dump, really) in an output file, e.g.:
```
$ kodidb_check -l > t.lst
$ kodidb_check --list-founddup --pick < t.lst
```

## kodidb_check_replay

The `kodidb_check_replay` runs a sequence of `kodidb_check` commands, typically a cleanup and check recipe, and stores each result in a subdirectory. The script is configured to run:

```
	0 original database
	1 cleaned unreferenced file records (--delete-unref)
	2 fixed paths (--fix-paths)
	3 fixed found files pt 1 (--remap-foundone)
	4 fixed found files pt 2 (--remap-foundlike)
	5 run postfix
```

The `postfix` script is given as an example to manually enter path and/or file remapping. Add special sqlite3 queries here or commands imported from history.log to tweak the last stage of the fix. It is a good idea to end the process with a `kodidb_check --clean-roots`
and `kodidb_check --check`, to remove any dangling (empty) directories and check the database once more.

```
usage: kodidb_check_replay source_MyVideos119.db (original/soure database to run the replay on)
       kodidb_check_replay n (stage number to resume the replay on, overwriting results of n, n+1 etc. uses the database from stage n-1)
```

## Example
- On the "kodi" system: retrieve the MyVideos119.db database from the kodi box (from `.kodi/userdata/Databases`)
- Place the MyVideos119.db file on the "local" system and call it MyVideos119.db.org (that's your backup)
- On the "local" machine
  - Make sure the local machine has access to your media library filesystem
  - configure the `localroot` and `localsources` accordingly
  - configure the `kodiroot` directory to the root path that is used on the kodi machine. Look in `.kodi/userdata/sources.xml` on the kodi system for the path or run `sqlite3 MyVideos119.db.org <<< "SELECT c22 FROM movie"` on the local system to see how the kodi path is constructed and how it relates to the local file system.
- Run `kodidb_check -l` to check if the output makes sense.
- Run `kodidb_check_replay MyVideos119.db.org` to create the sequence.
- Check for errors in the stdout.log and stderr.log files in the output directories (`grep -r ERROR` does wonders).
- Review the results
- Adjust the recipe and rerun the recipe from a given stage, e.g. `kodidb_check_replay 3` to rerun everything from the `--remap-foundone` step and onwards.
  - Note: the file system scan (from `localsources`) has an effect on which files are found/not found/duplicate. When this is changed or when the file system has changed structure/contents, delete `fileindex.fst` so it is rescanned next time and rerun from stage 3 to re-evaluate the found files.
- Tweak and repeat until you're happy with the outcome.
- On the "kodi" system:
  - Stop kodi by typing `systemctl stop kodi`
  - Copy the output MyVideos119.db database to the kodi system (in `.kodi/userdata/Databases`)
  - Start kodi by typing `systemctl start kodi`
  - Update the artwork with the great tool from https://github.com/MilhouseVH/texturecache.py by typing `texturecache.py c`
  - Depending on your use case, clean the video library from the user interface or by typing `texturecache.py vclean`. Or don't clean if you don't use that feature to keep your movies uptodate. Note: Clean Library deletes all movies that (still) can't be found on the file system, regardless whether it is '*in use*', so this may not be what you want.
  - Check in the Kodi UI if what you see is what you expect to see.

--
