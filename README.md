# Ansible role: borg

Installs and configures a borg client on EL and Debian based systems.

## Description

This role installs the Borg backup binary on a system and configures it to do
daily backups to a remote or local directory. Borg is a backup system written in
Python and supports authenticated encryption and deduplication. For more
information, see https://www.borgbackup.org/

Remote backups are executed via SSH and a (compatibe) Borg binary is expected
to be executable on the remote server. When backing up to a "local" directory,
that directory may be a mounted NFS or CIFS share. The backup is executed by
and configured for the root user.

Every backup client is by default backed up to a separate repository (which is not
positive for deduplication but prevends admins on one backup client to restore
files of another client). The progress of the backup is logged to /root/logs
on the backup client (see the brl\_borg\_combirepo variable).

Using this role should not require gathering of facts.

## Requirements

The precompiled binary should be downloaded from
https://github.com/borgbackup/borg/releases/ and placed in the *brl\_borg\_blobdir*
directory (see below). The version number should be added to the file name
(like *borg-linux64-1.1.10*). This requirement is in place because we can't
assume that the client has unlimited internet access and can download the binary
itself.

## Dependencies

None

## Example playbook

<pre>
---
- hosts:
    - all
  become: yes
  roles:
    - hamal03.borg
</pre>

## Role variables

### Summary

**brl\_borg\_use\_ssh**: access remote repository via ssh (default: True)<br/>
**brl\_borg\_version**: Borg binary version to download (default 1.1.10)<br/>
**brl\_borg\_blobdir**: Local directory where the binary can be found (default: /data/ansibleblobs)<br/>
**brl\_borg\_host**: remote host (required with ssh)<br/>
**brl\_borg\_user**: remote user (optional, recommended)<br/>
**brl\_borg\_port**: SSH port to use (default port 22)<br/>
**brl\_borg\_hostalias**: name to use in env & .ssh/config (default: borgbackupserver)<br/>
**brl\_borg\_encmode**: encryption mode (default repokey-blake2)<br/>
**brl\_borg\_password**: encryption password, mandatory when using encryption<br/>
**brl\_borg\_includes**: list of directories that should be included (default: [])<br/>
**brl\_borg\_excludes**: list of directories that should be excluded (default: [])<br/>
**brl\_borg\_backupall**: include OS directories (/bin. /usr, /lib\*, default false)<br/>
**brl\_borg\_path**: path part of repository without trailing short hostname (mandatory)<br/>
**brl\_borg\_realpath**: like path, but outside of a chroot (default brl\_borg\_path)<br/>
**brl\_borg\_combirepo**: if true, do not create "short hostname" subdirectories (default False)<br/>
**brl\_borg\_repogroup**: owning group of (remote) repository directory (default adm)<br/>
**brl\_borg\_cronhour**: what hour of the day dow we make backups (default 03)<br/>
**brl\_borg\_dailies**: number of daily backups to keep (default 7)<br/>
**brl\_borg\_weeklies**: number of weekly backups to keep (default 5)<br/>
**brl\_borg\_monthlies**: number of monthly backups to keep (default 12)<br/>
**brl\_borg\_yearlies**: number of yearly backups to keep (default 2)<br/>
**brl\_borg\_noprunes**: Do not remove old backups<br/>
**brl\_borg\_condscript**: full path to a script to run before backup<br/>

### Minimally needed variabled

* brl\_borg\_path
* brl\_borg\_host or brl\_borg\_use\_ssh set to "False"
* brl\_borg\_password or brl\_borg\_encmode set to "none"

### Elaboration on variables

#### brl\_borg\_use\_ssh
This boolean defines a backup to a remote server via SSH or to a local
directory. It defaults to using SSH.

#### brl\_borg\_version
Version of the borg binary to install. This role only installs the 64-bit
Linux version. The binary is expected in the "blobdir" (see below) and the
name needs to be "borg-linux64-*<version>*. If you want a different version
installed, download it from  https://github.com/borgbackup/borg/releases/,
preferably verify the download with PGP and copy it with the correct name to
the blobdir, then apply the version in this variable. The current default is
version 1.1.10.

#### brl\_borg\_configdir
Directory where the borg configuarion directory resides (defaults to 
/root/.config). If it is non-default, a symbilic link will be made to it named
/root/.config/borg.

#### brl\_borg\_blobdir
Directory on the ansible master node where the borg binary can be found
(default: /data/ansibleblobs)

#### brl\_borg\_host
Host to connect to using SSH to start the borg server. Required when using
SSH, ignored when using a local directory

#### brl\_borg\_user
The user that will run the remote borg server. The default is root but it is
recommended to use a dedicated user that has write access to the remote
repository.

#### brl\_borg\_port
TCP port to use for the remote SSH connection (default 22)

#### brl\_borg\_hostalias
A name to use in the BORG\_REPO environment variable and the /root/.ssh/config
file for the remote server. With this the correct configuration parameters for
the SSH connection and the correct SSH key are set. The SSH key that is used
only allows the remote session to start the "borg serve" command and limit its
access to only the directory with the borg repository.

#### brl\_borg\_encmode
The encryption mode that is used to encrypt the backups. If you use "none",
encryption is disabled. If encryption is enabled, it is done with AES256 in
CTR mode and the message authentication can be SHA256 or Blake-2b based. The
passphrase is used to encrypt a key blob that is either present in the
repository or only in the .config/borg directory. This is the case in when
using the "keyfile" or "keyfile-blake2" mode. If you use that, make sure to copy
the key file to somewhere safe in case the system crashes. Without it, you can't
restore data. The default is "repokey-blake2" which has the encrypted keyblob
in the repository and only relies on the passphrase for security.

#### brl\_borg\_password
The password to use for the backup encryption. It is ignored when the
"encmode" is not a "repokey" or "keyfile" variant and mandatory otherwise.<br/>
If you want to use a separate password for every backup client but still not
specify it every time, you could create a master passwword in a group variable
and use the hostname as a salt in a password hash to generate unique
passwords, like this:<br/>
<tt>"{{ my\_master\_password | password\_hash('sha256', inventory\_hostname) }}"</tt>

#### brl\_borg\_path
The path on the borg server where the backup repository is created. Below this
path, the default is that there is a subdirectory with the backup client's hostname as
directory name, so multiple backup clients can have the same "path" variable.
This behaviour can be changed with the brl\_borg\_combirepo option. This
variable is mandatory.

#### brl\_borg\_realpath
Like path, but if the path is inside a chroot environment, this gives the
complete path so Ansible can know where to create the repo. If not specified,
the value of the "brl\_borg\_path" variable is used.

#### brl\_borg\_combirepo
If true, no separate directory will be made based on the inventory\_hostename.
If multiple backup clients have the same "brl\_borg\_path" variable, this will
cause all of them to share the repository which can increase the deduplication.
Note that if encryption is enabled, all these backup clients should share the
encryption password. In that case you should also use either the "repokey" or
"repokey-blake2" value for the "brl\_borg\_encmode" variable since setting up the
the same local key blob on multiple backup clients is not trivial to automate.

#### brl\_borg\_repogroup
The unix group on the borg server that the files and directories of the borg
repo belong to. This group has read access to the backup files.

#### brl\_borg\_includes
A list of directories to include in the backup if they would be excluded
otherwise. The default list is empty.

#### brl\_borg\_excludes
A list of directories to exclude from the backup if they would be included
otherwise. The default list is empty. Includes are evaluated before excludes.

#### brl\_borg\_backupall
Include the operating system binary directories (/bin, /usr, /lib and /lib64).
By default, these directories are excluded. The default is false, which would
exclude the OS binary directories. Even though /usr is excluded by default,
/usr/local is still included.

#### brl\_borg\_cronhour
The backup is started via cron at this hour (default 3 AM) and a minute value
that is randomized to even out the various backups.

#### brl\_borg\_dailies
When pruning old backups, the number of daily backups to keep (default 7)

#### brl\_borg\_weeklies
When pruning old backups, the number of weekly backups to keep (default 5)

#### brl\_borg\_monthlies
When pruning old backups, the number of monthly backups to keep (default 12)

#### brl\_borg\_yearlies
When pruning old backups, the number of yearly backups to keep (default 2)

#### brl\_borg\_noprunes
Do not prune old backups (default false)

#### brl\_borg\_condscript
The full path to a script to run before the backup. If this script fails
(i.e., has a non zero exit code) the backup is aborted.

## License

This role is distributed under the Gnu Public License version 3 or later, except
the "autoborg.sh" script which is derived from the script in the borg software
distribution. The text of the GPL can be retrieved from the
[gnu.org](https://www.gnu.org/licenses/gpl-3.0.en.html) website. The license for
the autoborg.sh script is the following:

<pre>
Copyright (C) 2015-2019 The Borg Collective (see AUTHORS file)
Copyright (C) 2010-2014 Jonas Borgström <jonas@borgstrom.se>
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

 1. Redistributions of source code must retain the above copyright
    notice, this list of conditions and the following disclaimer.
 2. Redistributions in binary form must reproduce the above copyright
    notice, this list of conditions and the following disclaimer in
    the documentation and/or other materials provided with the
    distribution.
 3. The name of the author may not be used to endorse or promote
    products derived from this software without specific prior
    written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
</pre>

## Author

N: Rob Wolfram<br/>
E: propdf@hamal.nl<br/>
G: https://github.com/hamal03 <br/>
