# Kodi database management (MyVideos.db)

Utility to check, clean up and manage the kodi media player database (MyVideos119.db)

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

constraints: movies are not touched (these can be cleaned up with 'Clean Library' in kodi or 'texturecache.py vclean' on the kodi machine)
limitations: does not scan or check for musicvideos, episodes, tvshows
             tested on version 119 of MyVideos.db (Kodi 19)
             tested on linux-gnu and darwin21 (Mac OS X will require bash >=4 (for mapfile), which can be obtained from MacPorts or Brew)
```

