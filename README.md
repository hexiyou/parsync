# parsync
a parallel rsync wrapper in Perl

## NOTICE:  This project has been superceded by the parsyncfp tree at the same 
github: <https://github.com/hjmangalam/parsyncfp>

parsync is both buggier and slower than parsyncfp and is no longer being developed.

I'll leave the rest of the project intact, but it will not be further developed.
Happy to answer questions about either one <hjmangalam@gmail.com>

## Background
parsync is primarily tested on Linux, but (mostly) works on MacOSX
as well.

parsync needs to be installed only on the SOURCE end of the 
transfer and only works in local SOURCE -> remote TARGET mode 
(it won't allow remote local SOURCE <- remote TARGET, emitting an 
error and exiting if attempted). It requires that ssh shared keys be
set up prior to operation <https://goo.gl/ghCazV>.

It uses whatever rsync is available on the TARGET.  It uses a number 
of Linux-specific utilities so if you're transferring between Linux 
and a FreeBSD host, install parsync on the Linux side. 

The only native rsync options that parsync uses is '-a' (archive) &
'-s' (respect bizarro characters in filenames).  If you need to define 
the rsync options differently, then it's up to you to define ALL of 
them via '--rsyncopts' (the default '-a -s' flags will NOT be provided 
automatically.

parsync checks to see if the current system load is too heavy and tries
to throttle the rsyncs during the run by monitoring and suspending / 
continuing them as needed.

It uses the very efficient (also Perl-based) kdirstat-cache-writer
from kdirstat to generate lists of files which are summed and then
crudely divided into NP jobs by size.

It appropriates rsync's bandwidth throttle mechanism, using '--maxbw'
as a passthru to rsync's 'bwlimit' option, but divides it by NP so
as to keep the total bw the same as the stated limit.  It monitors and
shows network bandwidth, but can't change the bw allocation mid-job.
It can only suspend rsyncs until the load decreases below the cutoff.
(If you suspend parsync (^Z), all rsync children will suspend as well,
regardless of current state.)

Unless changed by '--interface', it tried to figure out how to set the 
interface to monitor.  The transfer will use whatever interface routing 
provides, normally set by the name of the target.  It can also be used for 
non-host-based transfers (between mounted filesystems) but the network 
bandwidth continues to be (usually pointlessly) shown.

[[NB: Between mounted filesystems, parsync sometimes works very poorly for
reasons still mysterious.  In such cases (monitor with 'ifstat'), use 'cp'
or 'tnc' (https://goo.gl/5FiSxR) for the initial data movement and a single
rsync to finalize.  I believe the multiple rsync chatter is interfering with 
the transfer.]]

It only works on dirs and files that originate from the current dir (or
specified via "--rootdir").  You cannot include dirs and files from
discontinuous or higher-level dirs.

### the ~/.parsync files 
The ~/.parsync dir contains the cache (*.gz), the chunk files (kds*), and the
time-stamped log files. The cache files can be re-used with '--reusecache'
(which will re-use ALL the cache and chunk files.  The log files are
datestamped and are NOT overwritten.

## Odd characters in names
parsync will sometimes refuse to transfer some oddly named files, altho 
recent versions of rsync allow the '-s' flag (now a parsync default) 
which tries to respect names with spaces and properly escaped shell 
characters.  Filenames with embedded newlines, DOS EOLs, and other 
odd chars will be recorded in the log files in the ~/.parsync dir.

** NB **
Because of the crude way that files are chunked, NP may be
adjusted slightly to match the file chunks. ie '--NP 8' -> '--NP 7'. 
If so, a warning will be issued and the rest of the transfer will be 
automatically adjusted.

** Debugging **
To see where you (or I) have gone wrong, look at the 
rsync-logfile\* logs in the .parsync dir.  If you're a masochist, use
the '--debug' flag which will spew lots of grotacious gratuitous info
to the screen.

## Options

[i] = integer number<br>
[f] = floating point number<br>
[s] = "quoted string"<br>
( ) = the default if any<br>


- --NP [i] (sqrt(#CPUs)) ... number of rsync processes to start optimal NP depends on many vars.  Try the default and incr as needed
- --startdir [s] (`pwd`) ... the directory it works relative to. If you omit it, the default is the CURRENT dir. You DO have to specify target dirs.  See the examples below.
- --maxbw [i] (unlimited) ... in KB/s max bandwidth to use (--bwlimit passthru to rsync). maxbw is the total BW to be used, NOT per rsync.
- --maxload [f] (NP+2)  ... max total system load - if sysload > maxload, sleeps an rsync proc for 10s
- --checkperiod [i] (5) ... sets the period in seconds between updates
- --rsyncopts [s] ... options passed to rsync as a quoted string (CAREFUL!) this opt triggers a pause before executing to verify the command.
- --interface [s] ... network interface to /monitor/, not nec use.<br>
    default: `/sbin/route -n | grep "^0.0.0.0" | rev | cut -d' ' -f1 | rev`<br>
    above works on most simple hosts, but complex routes will confuse it.<br>
- --reusecache ... don't re-read the dirs; re-use the existing caches
- --email [s] ... email address to send completion message (requires working mail system on host)
- --barefiles ... set to allow rsync of individual files, as oppo to dirs
- --nowait  ... for scripting, sleep for a few s instead of wait
- --version ... dumps version string and exits
- --help ... this help

## Examples

### Good Example 1
```% parsync  --maxload=5.5 --NP=4 --startdir='/home/hjm' dir1 dir2 dir3 hjm@remotehost:~/backups```

where:

- "--startdir='/home/hjm'" sets the working dir of this operation to '/home/hjm' and dir1 dir2 dir3 are subdirs from '/home/hjm'
- the target "hjm\@remotehost:~/backups" is the same target rsync would use
- "--NP=4" forks 4 instances of rsync
- "--maxload=5.5" will start suspending rsync instances when the 5m system load gets to 5.5 and then unsuspending them when it goes below it.

It uses 4 instances to rsync dir1 dir2 dir3 to hjm@remotehost:~/backups

###  Good Example 2
```% parsync --rsyncopts="-a -s --ignore-existing" --reusecache  --NP=3 --barefiles  *.txt   /mount/backups/txt```

where
-  "--rsyncopts='-a -s --ignore-existing'" are options passed to rsync
     telling it not to disturb any existing files in the target directory.
     '-a -s' are included bc use of --rsyncopts implies that the user
     defines ALL rsync options ('-a -s are default values, but blanked by
     the use of '--rsyncopts').
- "--reusecache" indicates that the filecache shouldn't be re-generated,
    uses the previous filecache in ~/.parsync
- "--NP=3" for 3 copies of rsync (with no "--maxload", the default is 4)
- "--barefiles" indicates that it's OK to transfer barefiles instead of
    recursing thru dirs.
- "/mount/backups/txt" is the target - a local disk mount instead of a network host.

It uses 3 instances to rsync *.txt from the current dir to "/mount/backups/txt".


### Error Example 1

```
% pwd
/home/hjm  # executing parsync from here

% parsync --NP4 --compress /usr/local  /media/backupdisk
```

Why this is an error:

- '--NP4' is not an option (parsync will say "Unknown option: np4")
    It should be '--NP=4'
- if you were trying to rsync '/usr/local' to '/media/backupdisk', 
    it will fail since there is no /home/hjm/usr/local dir to use as 
    a source. This will be shown in the log files in 
    ~/.parsync/rsync-logfile-<datestamp>_#
    as a spew of "No such file or directory (2)" errors
- the '--compress' is a native rsync option, not a native parsync option.
    You have to pass it to rsync with "--rsyncopts='--compress'"

The correct version of the above command is:

```% parsync --NP=4  --rsyncopts='--compress' --startdir=/usr  local  /media/backupdisk```

### Error Example 2
```% parsync --start-dir /home/hjm  mooslocal  hjm\@moo.boo.yoo.com:/usr/local```

Why this is an error:

- this command is trying to PULL data from a remote SOURCE to a 
    local TARGET.  parsync doesn't support that kind of operation yet.
    
The correct version of the above command is:

```# ssh to hjm\@moo, install parsync, then:
% parsync  --startdir=/usr  local  hjm\@remote:/home/hjm/mooslocal```
