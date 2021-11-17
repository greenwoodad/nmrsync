# nmrsync

nmrsync is a bash script for periodic synchronizing of Bruker NMR data to a data server 
using [rsync](https://download.samba.org/pub/rsync/rsync.html). It takes an input file 
as an argument which provides information including paths, SSH aliases, and rsync 
options. The script searches for files that have been modified recently (< x days as 
specified in the input file), searching in folders up to the level of 
RemoteDataPath/(user)/nmr/(data set). For example, if a new spectrum 1/ or 2/ appears in
the data set folder, the data set is flagged for syncing, but if something deeper 
(e.g. a proc file in a pdata folder) is changed, it won't be flagged.  

Before syncing, the script can search for folders that are identically named except for 
case differences (which are unique in Linux but indistinguishable on Windows and Mac), as
well as for folders that end in a period (which is permissible on Linux and Mac but not 
on Windows). The names of these spectra are then placed in SkipFileOld and they are not 
synced. An email is instead sent to the NMR manager who can then manually change the 
folder names once the data has finished acquiring. This can also be accomplished 
automatically using nmrfolderfix (https://github.com/greenwoodad/nmrfolderfix/).

There is an option to perform a second rsync to a second local location by specifying 
a third path in the input file. This rsync is performed with different customizable 
options. In the example input file, the second rsync skips copying permissions, 
modification times, or group information which enables (at least on my system) 
copying to a mounted windows file share.

I personally run this as a cron job every five minutes as well as every week with a second 
input file to ensure data is still eventually transferred after network or power outages. 

## Prerequisites

This script requires a linux operating system with rsync. It has been tested in CentOS 6.8,
7.5 and x on the local side and CentOS 7.5, CentOS 5.1, and RHEL7.3 on the remote side. 
I've only tested this with Bruker NMR data, but future releases may be able to handle file 
structures generated by other instruments.

The email feature requires that the command *sendmail* is working on the machine running the script.

## Installing

```sh
git clone https://github.com/greenwoodad/nmrsync
```
or 

```sh
git clone https://(your github username)@github.com/greenwoodad/nmrsyc.git
```

followed by:
```sh
chmod +x ./nmrsync/nmrsync
```

## Getting Started

### Setting up password-less ssh logins to instrument machines

Because this script is intended to be run as a cron job, it is necessary to authorize the local
machine to access the remote machine(s) with password-less ssh login using * ssh keys. * Tutorials
are available here: 
* [How To Set Up Passwordless SSH Login](https://linuxize.com/post/how-to-setup-passwordless-ssh-login/)
* [OpenSSH Config File Examples](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

Briefly: 
1) On the machine you want to run the script and send emails from (as the user you want to do this as) run the command:

```sh
ssh-keygen -t rsa -b 4096
```

This will generate files ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub 

Press enter at the prompt "Enter passphrase (empty for no passphrase):" to skip passphrase generation.

2) Next, run this command (from the local machine) for each remote workstation:

```sh
ssh-copy-id remote_username@remote_ip_address
```
You will be prompted for the password for this remote workstation. 

If ssh-copy-id is not available, you should be able to run this instead:

```sh
cat ~/.ssh/id_rsa.pub | ssh remote_username@remote_ip_address "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

3) Last, add SSH aliases to your hosts file. In /etc/hosts, add entries:

```sh
IPAddress DomainName SSHAlias
```

for each remote workstation.

for example:
```sh
198.51.100.50     dmx500.chem.university.edu       DMX500
198.51.100.54     av400.chem.university.edu        AV400
198.51.100.59     neo400.chem.university.edu       NEO400
```
The SSHAliases here should be the same SSHAliases you enter in the nmrsync input file. 

You should now be able to SSH to the remote workstations without entering a password by typing: 

```sh
ssh remote_username@SSHAlias
```

in addition to 

```sh
ssh remote_username@IPAddress 
```
and

```sh
ssh remote_username@DomainName 
```
The first time you do this, you will need to type "yes" to the question "Are you sure you want
to continue connecting (yes/no)?" however. After this, you will be able to run the script 
automatically without manual password entry.

### Configuring the input file

In the input file (nmrsync_input) there are a number of parameters and paths to set:

* `ScriptsPath`: Location of the main script and the input, emailtxt, and log folders on local machine.

* `SendmailPath`: Location of the sendmail application on local machine, probably /usr/sbin.

* `ManagerEmail`: Email address of the NMR facility manager.

* `AgeDay`: How many days back to look for recent experiments to sync. Default is 3, which works well unless you run spectra that take longer than 3 days to acquire.

* `RsyncOptions_1`: Rsync options for first rsync. Default is '-auvr --protect-args'

* `RsyncOptions_2`: Rsync options for second rsync (optional). Default is '-uvrlD --protect-args' 

* `SkipFlag`: Defines what folders are not synced. 'period' to skip folders ending in a period, 'dup' to skip folders with case-insensitive duplicates, 'both,' or 'none.' Default is 'both.' Note that if a different value of SkipFlag is specified with -s when the script is run, it overrules the value specified in the input file.			

* `Instrument`: Name of instrument. Can be anything (no spaces) but make sure it is unique (not entered twice in the table). 

* `SSHAlias`: Alias for password-less SSH to this instrument computer.

* `RemoteUser`: User on the remote computer that you can SSH as.
	
* `/nmr directory?`: Set this to 'y' for the default /(user)/nmr/(data set)/(expt #) data organization on the remote computer. Set it to 'n' for data organized as /(user)/(data set)/(expt #)

* `RemoteDataPath`: Full path containing NMR data on the remote comupter. Topspin/ICON-NMR usernames should be found in this folder.

* `LocalDataPath`: Full path on local computer to transfer the data to. 

* `MountedPath`: (optional) A second path on the local computer to transfer data to. This can be an external hard drive or a mounted windows file share, for instance. (There's no requirement that this actually refer to a mounted file system but it should be local.)

IMPORTANT: When editing this file, entries should be separated by either a tab or multiple spaces. 

Multiple input files can be prepared to run nmrsync in different configurations. 


NOTE: Additional modifications can be made to the variables 'ManualFlag', 'ExcludeFlag', and 'FullFlag' at top of the script itself. These are the default values for options that can be provided when the script is run.

## Usage

```sh
nmrsync [OPTIONS]... path/to/nmrsync_input
```

Options

-h, -?, --help                           Show help message.

-i, --input                              Set input file (flag optional).

-s, --skip (default 'both')              Set to 'period' to skip folders ending in period, 
                                         'dup' to skip case-insensitive duplicates, 'both' 
                                         to skip both and 'none' to skip none.

-m, --manual (default n)                 Set to y to operate in manual mode (enter password 
                                         instead of using SSH keys--not recommended).
										 
-e, --excludelist (default y)            Set to n to not utilize instrument-specific exclude 
                                         list in the input folder (will copy processed data).
										 
-f, --full (default n)                   Set to y to copy over all data instead of just 
                                         recently added data."

The defaults here can be modified at the top of the script itself.

To run this as a cron job, make an entry in your crontab like this:

```sh
*/5 * * * * /path/to/nmrsync "/path/to/input/nmrsync_input"
40 6 * * 0 /path/to/nmrsync "/path/to/input/nmrsync_input_weekly"
```

In the preceding example, there is a second input file which is configured to run looking for data that has been collected over the last week. The "fast" version is set to run every five minutes ( * /5) while the "slow" version is set to run on Sunday (0) at 6:40 AM (40 6).

## Contributing
Pull requests are welcome. 

## Authors

  - **Alex Greenwood** - *provided script* -
    [Greenwoodad](https://github.com/Greenwoodad)

## License
[MIT](https://choosealicense.com/licenses/mit/)
