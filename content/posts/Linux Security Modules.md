---
title: "Linux Security Modules"
date: "2020-03-05"
authors:
  - Will Shand
draft: false
tags:
  - linux
---

_Disclaimer: I threw these notes together for CU Boulder's Collegiate Cyber Defense Competition team for spring 2020. There may be some inaccuracies, and parts of the notes are still incomplete._


## Introduction {#intro-to-lsm}

The Linux kernel has a built-in _discretionary access control_ (DAC) system for controlling system resources. DACs are characterized by the fact that the owner of a resource may grant somebody else access to that resource; for instance, in Linux, the owner of a file could run `chmod +rwx myfile` to give every user on the system read, write, and execute permissions to the file. As a result, a process can privesc if it's able to run commands as a user with higher permissions.

[Linux Security Modules](https://en.wikipedia.org/wiki/Linux%5FSecurity%5FModules) (LSM) is a framework for implementating _mandatory access control_ (MAC) in Linux. Under a MAC policy, users cannot modify the permissions of resources; the permissions are set in stone by the system administrator. This makes privilege escalation significantly more difficult. For instance, say an attacker manages to get a reverse shell into an Apache webserver, and is eventually able to login as root. Even as the root user, the attacker is confined by whatever permissions are granted to the webserver process, which may be extremely restrictive.

There are a few other general characteristics of LSMs that differentiate them from traditional Linux DAC:

-   LSMs focus on assigning permissions to _processes_, whereas the Linux DAC policy assigns permissions to users and groups.
-   LSMs are generally "deny by default": if access to a resource isn't explicitly granted, then it's denied.
-   LSMs can have much more fine-grained permissions than Linux's DAC. For instance, SELinux doesn't just control access to files; you can define permissions over ports, syscalls, regions of memory, and more.

<hr>


## Logging and auditd {#logging-and-auditd}


### Introduction {#lsm-logging-intro}

`auditd` is an auditing daemon that logs system events and provides various utilities for searching logs. It is extremely useful for SELinux as it can be used to record denial to various resources.

`auditd` logs generally look something like this:

```-n
time->Sun Mar  1 00:11:43 2020
type=PROCTITLE msg=audit(1583046703.755:2547): proctitle=636174002F6574632F736861646F77
type=SYSCALL msg=audit(1583046703.755:2547): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=7ffeef4fae7b a2=0 a3=0 items=0 ppid=11335 pid=11336 auid=4294967295 uid=33 gid=33 euid=33 suid=33 fsuid=33 egid=33 sgid=33 fsgid=33 tty=(none) ses=4294967295 comm="cat" exe="/usr/bin/cat" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1583046703.755:2547): avc:  denied  { open } for  pid=11336 comm="cat" path="/etc/shadow" dev="sda1" ino=277524 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:shadow_t:s0 tclass=file permissive=1
type=AVC msg=audit(1583046703.755:2547): avc:  denied  { read } for  pid=11336 comm="cat" name="shadow" dev="sda1" ino=277524 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:shadow_t:s0 tclass=file permissive=1
```

These logs are fairly verbose, but there are a few pieces of information that are worth looking at:

-   `success` (under `type=SYSCALL`): tells us whether or not a given syscall was successful.
-   `comm` and `exe`: the command/executable that activated the rule.
-   `scontext`: the SELinux context of the resource that triggered the rule. Here, it's the context in which `/usr/bin/cat` is run.
-   `tcontext`: the SELinux context of the resource that the source was trying to access. Here, it's the context of `/etc/shadow`.


### `auditctl`: create `auditd` rules {#auditctl-create-auditd-rules}

`auditctl -w /etc/shadow -p rwa -k shadow_file`

-   Checks for reads, writes, and attribute changes to `/etc/shadow`
-   The `-k` flag assigns the key `shadow_file` to this rule

`auditctl -w /usr/local/bin -p x -k cron`

-   Checks for execution of binaries in the `/usr/local/bin` directory

`auditctl -a always,exit -F arch=b64 -F "auid>=1000" -S rename -S renameat -k rename`

-   Log attempts to rename a file using the `rename` or `renameat` syscalls
-   Only logs attempts for users with ID greater than or equal to 1000

Note that rules created with `auditctl` are lost after rebooting. If you want to make them permanent, you need to save them in `/etc/audit/rules.d/audit.rules`.

To delete a rule, use `-W` instead of `-w`.


### `ausearch`: search audit logs {#ausearch-search-audit-logs}

`ausearch -k shadow_file`

-   Search for logs with the `shadow_file` key

`ausearch --start recent -k cron`

-   Search for logs in the last ten minutes that are tagged with `cron`
-   You can also choose `today`, `yesterday`, etc. instead of `recent`.
-   You can specify a specific time or date too, e.g. `--start 03/01/2020` or `--start 03/01/2020 16:00:00`
-   There is an `--end` flag that, in conjunction with `--start`, allows you to specify the end of a time range.

`ausearch -se system_u:system_r:httpd_t:s0`

-   Search for all logs where either the source SELinux context or target SELinux context is `system_u:system_r:httpd_t:s0`

`ausearch -c apache2 -f /etc/shadow`

-   Search for all logs triggered by the `apache2` command related to the `/etc/shadow` file.
-   You can use `-x` instead of `-x` to specify the path to the executable instead of its name.


### `ausyscall`: view available syscalls {#ausyscall-view-available-syscalls}

`ausyscall --dump`

-   List all linux syscalls and their numeric codes

`ausyscall 4`

-   Find the name of the syscall corresponding to the ID `4`
-   Useful when reviewing `auditd` logs

<hr>


## SELinux {#lsm-selinux}


### Introduction {#selinux-intro}

SELinux is the default LSM enhancement for RHEL, Fedora, and CentOS. It is also installed in Debian, although as of Debian 10 (Buster) it is deactivated by default, with AppArmor enabled in its place. SELinux for Ubuntu has been largely unmaintained since Ubuntu 9.10 and broken since Ubuntu 12.04. Although it is still possible to install and use SELinux via the Debian repositories (where SELinux is still maintained), it is generally recommended that you use AppArmor for Ubuntu instead.


### Installation {#selinux-installation}

To get SELinux up and running, you probably want to install the following packages: `selinux-basics selinux-policy-default auditd`. In addition, depending on your system you may need to explicitly enable SELinux. You can check whether or not SELinux has been enabled by running `getenforce`; if you get the response `Disabled`, then you will need to explicitly activate it. Generally, this can be done by running `selinux-activate` and rebooting.


### SELinux utilities {#selinux-utilities}


#### `getenforce`, `sestatus`, and `seinfo`: get information about SELinux {#getenforce-sestatus-and-seinfo-get-information-about-selinux}

These three utilities each provide information about the current status and configuration of SELinux on your system:

-   `getenforce`: gets the current SELinux state, which can be "Disabled", "Permissive", or "Enforcing".
-   `sestatus`: shows the current SELinux state like `getenforce`, but also provides a little more information (e.g. the name of the loaded policy).
-   `seinfo`: get information about the current SELinux policy. Here are some example queries you can run with `seinfo`:
    -   `seinfo -b`: show all of the available SELinux booleans, and whether they are on or off.
    -   `seinfo -x -u staff_u`: show the full definition of the `staff_u` SELinux user. This will show you what roles are assigned to the `staff_u` user.
    -   `seinfo -x -r user_r`: show the full definition of the `user_r` SELinux role. This will show you what types are assigned to the `user_r` user.


#### `selinux-activate`: enable SELinux {#selinux-activate-enable-selinux}

Sets SELinux to activate after you reboot. Your machine will reboot into whatever mode has been assigned to the `SELINUX` variable in `/etc/selinux/config`.


#### `setenforce`: set SELinux state {#setenforce-set-selinux-state}

When SELinux is activated, you can use `setenforce` to change the current SELinux mode. `setenforce 0` will put SELinux in permissive mode, and `setenforce 1` will put it in enforcing mode.


#### `sesearch`: search SELinux policies {#sesearch-search-selinux-policies}

`sesearch` allows you to search for specific rules in your SELinux policy. Here are some examples of how to use it:

-   `sesearch --allow -s auditd_t -t auditd_log_t`: search for all `allow` rules with `auditd_t` as their source type and `auditd_log_t` as their target type.
-   `sesearch --allow -s httpd_t -p read,write`: find all rules with source type `httpd_t` that grant the `read` and `write` permissions.
-   `sesearch --allow -s httpd_t -t auditd_log_t -ds -dt`: find all rules that _exactly_ have `httpd_t` as their source type, and `auditd_log_t` as their destination type. Without the `-ds` and `-dt` flags, `sesearch` typically just matches by attribute contents.


#### `chcon`: change context of a file {#chcon-change-context-of-a-file}

Suppose that we want to change the SELinux context of `/var/www/html/index.html` to match that of the directory `/var/www/html/`. Running `ls -dZ /var/www/html`, we see that this directory has context `system_u:object_r:httpd_sys_content_t`. To change the context of `/var/www/html/index.html`, we use `chcon` as follows:

```nil
# chcon -u system_u -r object_r -t httpd_sys_content_t /var/www/html/index.html
```

But if you just want to change the type, you only have to use the `-t` flag, e.g. `chcon -t $selinux_type $target`

You can make things even easier by using the `--reference` flag:

```nil
# chcon --reference /var/www/html/ /var/www/html/index.html
```

This gives `/var/www/html/index.html` the same SELinux context as the `/var/www/html/` directory.


#### `restorecon`: restore context of a file {#restorecon-restore-context-of-a-file}

Recursively restore context of all files in `/var/www/html` to have the same context as the `/var/www/html/` directory:

```nil
# restorecon -R /var/www/html
```

You can use the `-v` flag for verbose output.


#### `semanage`: main SELinux configuration tool {#semanage-main-selinux-configuration-tool}

Can be used to manage SELinux settings for

-   node
-   file context
-   booleans
-   permissive state
-   dontaudit

Unlike `chcon` or `restorecon`, rules set via `semanage` are saved so that they become reapplied after reboot.

Examples:

-   `semanage fcontext -l`: List all of the SELinux policy rules around file contexts.
-   `semanage fcontext -a -t httpd_sys_content_t "/foo(/.*)?"`: add the type `httpd_sys_content_t` to the `/foo` directory and all of its contents.
-   `semanage fcontext -a -e /var/www/html /foo`: add a rule to make the context of `/foo` match that of `/var/www/html`. After this, you can run `restorecon -vR /foo` to apply the new label to `/foo`.


#### `audit2why` {#audit2why}


#### `audit2allow`: generate SELinux policies from logs of denied operations {#audit2allow-generate-selinux-policies-from-logs-of-denied-operations}

`audit2allow` uses log files (`/var/log/audit/audit.log` if `auditd` is installed, and `/var/log/messages` otherwise) to generate SELinux rules. It's an extremely useful tool for fixing broken policies.

For example, suppose you run `ip addr`, and it appears to be denied by SELinux (a situation I encountered on Debian 10). To figure out where the issue is, let's start by reviewing the audit logs:

```nil
# ausearch --start recent
..
time->Wed Mar  4 21:20:21 2020
type=AVC msg=audit(1583360421.558:1732): avc:  denied  { signull } for  pid=305 comm="systemd-journal" scontext=system_u:system_r:syslogd_t:s0 tcontext=system_u:system_r:NetworkManager_t:s0 tclass=process permissive=0
```

We can immediately make a few notes about the log:

-   The source context is `system_u:system_r:syslogd_t:s0`
-   The target context is `system_u:system_r:NetworkManager_t:s0`
-   The command that triggered the denial was `systemd-journal`

Overall, it looks like the command was denied because `systemd-journal` sent `SIGNULL` to a process with a different context, and SELinux blocked it. We can use `sesearch` to try and see what rules are defined for `syslogd_t` and `NetworkManager_t`:

```nil
# sesearch -s syslogd_t -t NetworkManager_t -A
allow syslogd_t domain:dir { getattr ioctl lock open read search };
allow syslogd_t domain:file { getattr ioctl lock open read };
allow syslogd_t domain:lnk_file { getattr read };
allow syslogd_t domain:process getattr;
```

It looks like there aren't any rules directly related to both `syslogd_t` and `NetworkManager_t`. We're going to need to add a new rule that allows processes of type `syslogd_t` to send `SIGNULL` to processes of type `NetworkManager_t`.

To do this, we use the `audit2allow` command. First, we use `audit2allow` to give us more information about how to fix this issue:

```nil
# sudo ausearch -se system_u:system_r:NetworkManager_t | tail -n 1 | audit2allow -w
type=AVC msg=audit(1583407237.188:26): avc:  denied  { search } for  pid=478 comm="NetworkManager" name=".cache" dev="vda1" ino=410827 scontext=system_u:system_r:NetworkManager_t:s0 tcontext=system_u:object_r:xdg_cache_t:s0 tclass=dir permissive=1

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```

With the `-a` flag, `audit2allow` will tell us what rule needs to be added in order to fix the error:

```nil
# sudo ausearch -se system_u:system_r:NetworkManager_t | tail -n 1 | audit2allow -a


#============= NetworkManager_t ==============
allow NetworkManager_t xdg_cache_t:dir search;
```

(**Note**: you can pipe in multiple logs at once in order to generate multiple SELinux rules.)

Finally, we create a new policy module with the rule:

After running `semodule -i syslogd.pp` and rebooting, a new SELinux rule will be added to your policy that permits the previously forbidden action.

```nil
# sudo ausearch -se system_u:system_r:NetworkManager_t | tail -n 1 | audit2allow -a -M syslogd
 ************************** IMPORTANT ******************************
To make this policy package active, execute:

semodule -i syslogd.pp
```


### Additional references {#selinux-addtl-refs}

-   [SELinux on the Gentoo Linux wiki](https://wiki.gentoo.org/wiki/SELinux)
-   [Quick intro to SELinux](https://wiki.gentoo.org/wiki/SELinux/Quick%5Fintroduction)
-   _The SELinux Notebook, 4th edition_.
-   [The RHEL SELinux documentation](https://access.redhat.com/documentation/en-us/red%5Fhat%5Fenterprise%5Flinux/6/html/security-enhanced%5Flinux/index)

<hr>


## AppArmor {#apparmor}


### Introduction {#apparmor-intro}

AppArmor is another Linux Security Module for implementing MAC that is enabled by default on Ubuntu and recent Debian distributions. Its chief goal is to be more user-friendly than SELinux.

Whereas SELinux is label-based, AppArmor is path-based: access to files and executables is based on their path in the filesystem. In addition, AppArmor rules are defined in profiles (e.g. `/etc/apparmor.d/usr.sbin.apache2`), in contrast to SELinux's concept of a system-wide policy. In order to restrict a binary, we must define a new profile for it, and then put that profile in enforcing mode.


### Installation {#apparmor-installation}

AppArmor is enabled by default on Ubuntu since version 7.10, and on Debian since Debian 10 (Buster). However, like SELinux, AppArmor has a few packages with convenient files and utilities that you should install if you want to use it:

-   `apt install apparmor-profiles apparmor-easyprof apparmor-utils`


### Using AppArmor {#using-apparmor}

In this section, we'll create an AppArmor profile for the `apache2` binary using the following steps:

1.  Create a template profile for the binary using `aa-easyprof`;
2.  Load that profile using `apparmor_parser`;
3.  Put the profile in complain mode using `aa-complain`;
4.  Add new AppArmor rules using `aa-logprof`; and
5.  Enforce the profile with `aa-enforce`.

Note that you may not necessarily need to go through all of these steps, e.g. if you already have a default profile for the binary. There are also some tools that wrap a few of these steps together, such as `aa-genprof`.


#### 1. Create a template `apache2` profile {#1-dot-create-a-template-apache2-profile}

Call `aa-easyprof /usr/sbin/apache2 > /etc/apparmor.d/usr.sbin.apache2`. The newly created profile will be mostly empty; it may look something like this:

```nil
/etc/apparmor.d/usr.sbin.apache2:

#include <tunables/global>

# No template variables specified

"/usr/sbin/apache2" {
  #include <abstractions/base>
  # No abstractions specified
  # No policy groups specified
  # No read paths specified
  # No write paths specified
}
```


#### 2. Load the profile {#2-dot-load-the-profile}

Now call `apparmor_parser /etc/apparmor.d/usr.sbin.apache2` to read the new profile. Note that if you've previously loaded in `/etc/apparmor.d/usr.sbin.apache`, you'll need to reload it by adding the `-r` flag.


#### 3. Put the profile in complain mode {#3-dot-put-the-profile-in-complain-mode}

Put the binary in complain mode by calling `aa-complain apache2`. In complain mode, AppArmor will scan log files to try to identify what capabilities the `apache2` binary needs to have.

Visit your Apache webserver, and start trying to simulate how it would be used in real life. As you test the webserver more and more, AppArmor will get an increasingly good idea of what capabilities the server is using.


#### 4. Add new AppArmor rules using `aa-logprof` {#4-dot-add-new-apparmor-rules-using-aa-logprof}

After you've tested your webserver sufficiently, run `aa-logprof`. This will read various logfiles in `/var/log` to see what files the Apache webserver accessed, and what capabilities it used. `aa-logprof` will then present you with a list of capabilities, which you can choose to accept, deny, ignore, and so on. Choose whichever capabilities your webserver needs; any capabilities that are not explicitly granted to the webserver will be denied.


#### 5. Enforce the profile {#5-dot-enforce-the-profile}

Call `aa-enforce apache2` to put the profile in enforcing mode. AppArmor is now running, and will deny capabilities not granted in `/etc/apparmor.d/usr.sbin.apache2`.


### Customizing profiles {#customizing-profiles}

It is often desirable to hand-edit your AppArmor profiles to make them a little more specific. Here are some general syntax rules for AppArmor profiles:

**Deny rules**: To write a deny rule, you just write `deny /path/to/file perms`.

-   `deny /etc/shadow rw`: deny read and write access to `/etc/shadow`.
-   `deny /etc/sudo x`: deny execute permission to `/etc/sudo`.

Note that AppArmor is deny-by-default, so you don't generally need to explicitly state every resource that you want to deny access to.

**Accept rules**:

**Tunables**: where possible, you should avoid using explicit paths like `/home/`, and instead use tunables, which are configurable variables defined in `/etc/apparmor.d/tunables`. For instance, instead of writing a rule like `/home/*/ r`, you should use `@{HOME}/ r`.


### Debugging AppArmor {#debugging-apparmor}


### Additional references {#apparmor-addtl-refs}

-   [AppArmor documentation](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation)
-   [A tutorial on creating a new AppArmor profile](https://ubuntu.com/tutorials/beginning-apparmor-profile-development)
-   You can download a high-level [whitepaper](https://ubuntu.com/engage/apparmor-intro) on AppArmor from Canonical's site.


## Miscellaneous references {#miscellaneous-references}

-   The [_Linux Security Module Framework_ paper](https://www.kernel.org/doc/ols/2002/ols2002-pages-604-617.pdf) that first introduced LSM.

