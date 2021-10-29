# nmrsync

nmrsync is a bash script for periodic synchronizing of Bruker NMR data to a data server 
using [rsync](https://download.samba.org/pub/rsync/rsync.html). It takes an input file as 
an argument which provides information including paths, SSH aliases, and settings regarding 
how frequenty the rsync is to be performed. The script searches for files that have been 
modified recently (< x min or days old, specified in the input file), searching in folders 
up to the level of $RemoteDataPath/user/nmr/experiment. For example, if a new spectrum 1/ or 2/ appears 
in an experiment folder, the experiment is flagged for syncing, but if something deeper (e.g. 
a proc file in a pdata folder) is changed, it won't be flagged. 

It operates in two modes, "fast" and "slow," to be specified in the input file. In the "fast" 
mode it maintains the 5 most recent spectra in a list ("$SyncFileOld") and continues to sync 
these until they are displaced by newer spectra -- this allows for syncing of spectra that take 
longer than $AgeMin to acquire and are therefore not "new" by the time the spectrum is done. 

Before syncing, the script can search for folders that are identically named except for case 
differences (which are unique in Linux but indistinguisable on Windows and Mac), as well as for 
folders that end in a period (which is permissible on Linux and Mac but not on Windows). The 
names of these spectra are then placed in $SkipFileOld and they are not synced. An email is 
instead sent to the NMR manager who can then manually change the folder names once the data 
has finished acquiring. 

There is an option to perform a second rsync to a second local location by specifying a third 
path in the input file. This rsync is performed with different customizable options copying permissions, modification times, 
or group information which enables (at least on my system) copying to a mounted windows file share.

I personally run this as a cron job every five minutes as well as every week with a second input file to ensure data is still eventually transferred after network or power outages. 

## Prerequisites

This script requires a linux operating system with rsync. It runs in CentOS 6.8, 7.5 and x on the 
local side and CentOS 7.5, CentOS 5.1, and RHEL7.3 on the remote side. I've only tested this with 
Bruker NMR data, but future releases may be able to handle file structures generated by other instruments.

The email feature requires that the command *sendmail* is working on the machine running the script.

## Installing

git clone https://github.com/greenwoodad/nmrsync

## Getting Started

### Setting up password-less ssh logins to instrument machines

Because this script is intended to be run as a cron job, it is necessary to authorize the local
machine to access the remote machine(s) with password-less ssh login using * ssh keys *. Tutorials
are available here: 
* [How To Set Up Passwordless SSH Login](https://linuxize.com/post/how-to-setup-passwordless-ssh-login/)
* [OpenSSH Config File Examples](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

Briefly: 
1) On the machine you want to run the script and copy data to (as the user you want to do this as) run the command:

```sh
ssh-keygen -t rsa -b 4096
```

This will generate files ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub 

Press enter at the prompt "Enter passphrase (empty for no passphrase):" to skip passphrase generation.

2) Next, run this command for each remote workstation:

```sh
ssh-copy-id remote_username@remote_ip_address
```
You will be prompted for the password for this remote workstation. 

If ssh-copy-id is not available, you should be able to run this instead:

```sh
cat ~/.ssh/id_rsa.pub | ssh remote_username@remote_ip_address "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

3) Last, add SSH aliases to your hosts file. In /etc/hosts, add entries:

IPAddress DomainName SSHAlias

for each remote workstation.

for example:
198.51.100.50     dmx500.chem.university.edu       DMX500
198.51.100.54     av400.chem.university.edu        AV400
198.51.100.59     neo400.chem.university.edu       NEO400

The SSHAliases here should be the same SSHAliases you enter in the input file. 

You should now be able to SSH to the remote workstations without entering a password by typing: 

```sh
ssh SSHAlias
```

The first time you do this, you will need to type "yes" to the question "Are you sure you want
to continue connecting (yes/no)?" however. After this, you will be able to run the script 
automatically without manual password entry.


### Configuring the input file

In the input file (nmrsync_input) there are a number of parameters and paths to set:

* `ScriptsPath`: Location of the main script and the input, emailtxt, and log folders on local machine.

* `SendmailPath`: Location of the sendmail application on local machine, probably /usr/sbin.

* `ManagerEmail`: Email address of the NMR facility manager.

* `FastMode`: y or n-- y to run in "fast" mode looking for experiments run recently (e.g. within ~60 min) .

* `AgeDay`: Used in "slow" mode, how many days back to look for recent experiments to sync.

* `AgeMin`: Used in "fast" mode, how many minutes back to look for recent experiments to sync.

* `Instrument`: Name of instrument. Can be anything (no spaces) but make sure it is unique (not entered twice in the table). 

* `SSHAlias`: Alias for password-less SSH to this instrument computer.

* `RemoteUser`: User on the remote computer that you can SSH as.

* `RemoteDataPath`: Path containing NMR data on the remote comupter. Topspin/ICON-NMR usernames should be found in this folder.

* `LocalDataPath`: Path on local computer to transfer the data to. 

* `MountedPath`: (optional) A second path on the local computer to transfer data to. This can be an external hard drive or a windows file share, for instance. (There's no requirement that this actually refer to a mounted file system but it should be local.)

IMPORTANT: When editing this file, do not use entries that contain spaces. 

Multiple input files can be prepared to run nmrsync in different configurations. 

## Usage

```sh
nmrsync [OPTIONS]... path/to/nmrsync_input
```

Options

-h, -?, --help                           Show help message.

-i, --input                              Set input file (flag optional).

-m, --manual (default n)                 Set to  to operate in manual mode (enter password 
                                         instead of using SSH keys--not recommended).
										 
-e, --excludelist (default y)            Set to n to not utilize instrument-specific exclude 
                                         list in the input folder (will copy processed data).
										 
-f, --full (default n)                   Set to y to copy over all data instead of just 
                                         recently added data."
										 
-ra1, --rsync_add_1 (default '-a')       Provide additional options for the first rsync.

-ra2, --rsync_add_2 (default '-rlD')     Provide additional options for the second rsync. 

To run this as a cron job, make an entry in your crontab like this:

*/5 * * * * /path/to/nmrsync "/path/to/input/nmrsync_input"
40 6 * * 0 /path/to/nmrsync "/path/to/input/nmrsync_input_weekly"

In the preceding example, there is a second input file which is configured to run in "slow" mode, looking for data that has been collected over the last week. The "fast" version is set to run every five minutes (*/5) while the "slow" version is set to run on Sunday (0) at 6:40 AM (40 6).

## Contributing
Pull requests are welcome. 

## Authors

  - **Alex Greenwood** - *provided script* -
    [Greenwoodad](https://github.com/Greenwoodad)

## License
[MIT](https://choosealicense.com/licenses/mit/)
