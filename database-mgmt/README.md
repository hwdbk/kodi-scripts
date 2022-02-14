# Kodi database management (MyVideos119.db)

Kodi is a great media player. It is even great at finding metadata around movies and the like (called scraping) and storing/displaying this data and images in the user interface. What Kodi doesn't do very well (read: doesn't do at all), is keeping track of your media files if you hapen to move these around (and don't we all do that, keeping the library organised?). In Kodi, there's only two things you can do: Clean the Library (which will *delete* all entries that are no longer in the expected location on the disk, even if you just moved or renamed the original media file), and Rescrape (which will re-download all metadata from one of the online sources).

There are many 'howtos' around on managing file and directory paths in a MyVideos119.db. None of these (including the https://kodi.wiki/view/HOW-TO:Update_SQL_databases_when_files_move or https://kodi.wiki/view/HOW-TO:Update_Paths_In_MySQL) work(ed) very well for various reasons. Catching all variations and multiple occurrences of file paths, different kodi path notation, URL-encoded- and XML-escaped- (if you use the XML export) and UTF-8 strings (like Antonín Dvořák), then at the same time make sure the database remains consistent is not trivial and can't be caught in a simple 'search-replace' or a couple of SQL queries - that's why I started writing this tool, in need for some granular control over the process and being able what happend, what I screwed up and retrace my steps in the next run.

##kodidb_check

`kodidb_check` is a utility to check, clean up and manage the kodi media player database (MyVideos119.db)

This script provides a number of options to check, fix and clean up a Kodi MyVideos database. It can, among others:
- remap a path or a file, and make sure all referenced files are updated properly
- rename a file, path, or part of a path and make sure all referenced files are updated properly
- do a consistency check on the directory structure as registered in the database, and fix it
- list duplicate files found in the database, and optionally provide a prompt to fix the ambiguity
- list files from the database that can no longer be found (located) on the local file system, and optionally provide a prompt to fix the missing items
- save all database-modifying operations in a log file (`history.log`), so you can review them or for record-keeping. The commands are printed in a bash-ready form so you and even extract them for replay or save them in a script (you need to run `cut -f 2 history.log` to strip the dates).

Definitions:
  - "kodi" is referred to as the machine you run kodi on. This can be a linux computer, an embedded linux player (OpenELEC, CoreELEC etc.) or MacOSX.
  - "local" is the machine you run these scripts on. This can be a linux computer or MacOSX. Theoretically, it could also be the embedded linux player, but I found the bash there not capable of using arrays.
- The script works off the concept of (media) files. The files have a path (directory) where they are stored. Files can be '*in use*' and therefore important to you and for the script to manage. *In use* means:
  - The file is (used by) a kodi movie (NOTE: the script doesn't do TV Shows or Music Videos)
  - The file was played (has a play count > 0)
  - The file playback was in progress (this is called a 'bookmark')
- Files that *are* in use need to be treated carefully - the watched state, for instance, particularly for files that are not movies, is an important indicator in the file browser interface in Kodi. The bookmark is important to allow you to resume viewing where you left off last time.
- Files that are *not* in use don't need to be kept in the database. Why would you? Over the last 7 years that I've used kodi, the database contracted around 35k files (and paths) that were just scanned at some time, then moved, removed, disappeared, but never played (in full or partial) or used by a movie - these can all be cleaned out. Finding and fixing the stuff that needs to be kept is the trick here. The script got me from:

```
$ kodidb_check -l > dblist.lst
$ kodidb_check --summary < dblist.lst
f_founddup______: 97
f_foundlike_____: 10587
f_foundone______: 2769
f_hasbookmark___: 1289
f_hasmovie______: 1031
f_hasplaycount__: 3317
f_isunref_______: 34945
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
- Set up the `localroot` and `kodiroot` first in the script: they should point to the highest order directory that the kodi box shares
with the local machine. The idea is that kodi has a certain view on the file system that holds the media, and the local machine has the same view, but a different root path. localroot and kodiroot bring these together, and, if necessary, also takes protocol into account. Kodi, for instance, can use paths that start with `smb://x.x.x.x/` and the local machine can have the media directory mounted on `/mnt/media/`, or `/Volumes/media` if you own a Mac. This will be different for every setup, hence the two variables `localroot` and `kodiroot`.
- Secondly, set up the `localsources` variable - these is the list of local path(s) where the media can be found (in the script, `/mnt/media/video/movies /mnt/media/video/incoming` are used as an example, add more or less if necessary). The script will scan these directory trees to crossreference against the database, so this can take a little while. The result is stored in `fileindex.fst`, so only scanned once (delete this file if the `localsources` contents have changed and it will be re-scanned).
- The script does not touch (write/modify) your filesystem. It does read the file system to find files and check if (media) files and directories are there. This is to make sure the database, once put back on the kodi box, still works. If you're paranoid, you can mount the file system r/o and see that the script still works.
- I have no idea how the tool works if you have several removable disks with media; in theory, you could run the script several times and set up the `localroot` for each removable disk, and run the script.
- The script *does* modify the MyVideos119.db database. Always make a backup copy before attempting a cleanup. Hint: call the backup copy something different, like MyVideos119.db.org, so it can not accidentally be used by `kodidb_check`. Use the `kodi_check_replay` script to explicitly store and work on separate phases of the cleanup, with multiple backup points to retrace your steps.

Constraints:
- Movies are not touched (these can be cleaned up with 'Clean Library' in kodi or 'texturecache.py vclean' on the kodi machine)
- Does not scan or check for musicvideos, episodes, tvshows, sorry... That's because I don't use these types of media. If you use these, they do not classify the scanned files as '*in use*' and the files may be cleaned up. If you despereately need this feature, drop me a line.
- The script works off the MyVideos database, not NFO files that may be stored alongside the media.

Limitations:
- The 'kodi' system: tested on version 119 of MyVideos.db, from a Kodi 19 'Matrix' running on an embedded linux player (so it has / in the kodi paths, not \). I have no clue what a database looks like on a Windows kodi box.
- The 'local' system: tested on linux-gnu and darwin21 

Dependencies:
- The script uses sqlite3, which is the format of the kodi databases.
- The script relies on bash >=4 (for the mapfile builtin function). MacOSX 'local' machines may need to install bash and/or sqlite3 from MacPorts or Brew.
- The script uses mac2syn (https://github.com/hwdbk/synology-scripts/blob/master/mac-nfd-conversion/mac2syn) for converting MacOSX NFD (normalization form decomposed) UTF-8 strings to 'normal' UTF-8. The invocaiton is 'iffed' around an OSTYPE check, so if you're not running MacOSX as a local machine, it should not be a problem.

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
              --clean-root                       cleanup the directory structure starting from the root dir in the database, insofar paths are not used
              --remap-foundone                   remap all read 'foundone' files to the exact match listed in the log
              --remap-foundlike                  remap all read 'foundlike' files to the almost exact match listed in the log (matched without extension)
              --list-founddup [--pick]           list the read duplicate matches in a more readable form and, with --pick, present a prompt where the user can pick any of the found duplicates
              --list-notfound [--pick]           list all read 'notfound' files and, with --pick, present a prompt remap the file interactively. use this if you can drop file paths on the prompt
              --rename-founddup oldstr newstr    remap files with duplicate matches after substituting oldstr with newstr in the full path
              --rename-notfound oldstr newstr    remap files which weren't found after substituting oldstr with newstr in the full path
```

`kodidb_check_replay`: utility to run a recipe of `kodidb_check` commands, and store each intermediate result in a subdirectory

The `kodidb_check_replay` runs a sequence of `kodidb_check` commands, typically a cleanup and check recipe, and stores each result in a subdirectory. The script is configured to run:

```
	0 original database
	1 cleaned unreferenced file records (--delete-unref)
	2 fixed paths (--fix-paths)
	3 fixed found files pt 1 (--remap-foundone)
	4 fixed found files pt 2 (--remap-foundlike)
	5 run postfix
```

The `postfix` script is given as an example to manually enter path and/or file remapping. Add special sqlite3 queries here or commands imported from history.log to tweak the last stage of the fix. It is a good idea to end the process with a `kodidb_check --clean-root`
and `kodidb_check --check`, to remove any dangling (empty) directories and check the database once more.

```
usage: kodidb_check_replay source_MyVideos119.db (original/soure database to run the replay on)
       kodidb_check_replay n (stage number to resume the replay on, overwriting results of n, n+1 etc. uses the database from stage n-1)
```

Example:
- On the "kodi" system: retrieve the MyVideos119.db database from the kodi box (in `.kodi/userdata/Databases`)
- Place the MyVideos119.db file on the "local" system and call it MyVideos119.db.org (that's your backup)
- On the "local" machine
  - Make sure the local machine has access to your media library filesystem
  - configure the `localroot` and `localsources` accordingly
  - configure the `kodiroot` directory to the root path that is used on the kodi machine. Look in sources.xml for the path or run `sqlite3 MyVideos119.db.org <<< "SELECT c22 FROM movie"` to see how the path is constructed and relates to the local file system.
- Run `kodidb_check -l` to check if the output makes sense.
- Run `kodidb_check_replay MyVideos119.db.org` to create the sequence.
- Check for errors in the stdout.log and stderr.log files in the output directories (grep -r ERROR does wonders).
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

--
