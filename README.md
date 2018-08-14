# SELinux for Web Nerds in a Hurry

# (Incomplete Draft)


This document is for web developers who need to quickly develop a working relationship with SELinux on RHEL or CentOS without investing the time to dive very far in to the details. If you have more than a few minutes to spare, I recommend that you take a look at the [resources](#resources) section for deeper, better, more expert guidance.

## Learning Objectives

* Scratch the surface of the core SELinux concepts:  Mndatory Access Control, contexts/labels, RHEL's `targeted` policy
* Learn the two most common cause of SELinux errors and their symptoms
* Learn the minimal toolbox for identifying and fixing those common errors

## What is SELinux?

SELinux is a tool that helps ensure that a Linux system only runs the software that its supposed to and that software only
operates on an approved set of resource. It does this by providing Linux with kernel-level Mandatory Access Control that is
applied in addition to (and after) the normal Linux permission system.

The basic components are of SELinux are:

* labels -- identifiers for system features (files, ports, processes, etc), also called contexts.
* a policy -- a set of rules for allowing system features to operate on other system features.  (May X do Y to Z?)

Labels are of the format `user:role:type:level`, but, for our purposes, we generally only have to care about the type part of
the label, which will usually look  something like `foo_t`.

## RHEL's `targeted` Policy

We're using RHEL/CentOS, so we'll mainly be working RedHat's `targeted` policy, which focuses on the most important, commonest,
and most exposed services that we will typically run.

As web developers, our code mostly runs inside established processes like `tomcat` or `apache`, so just about everything we
need already exists in the `targeted` policy. 

Everything else, including most things run by logged in users, is considered "unconfined" and given the label `unconfined_t`,
which is designed to provide minimally impactful protection from some common attacks. 

## Enforcing and Permissive Mode

SELinux can run in two modes:

* Permissive -- policy violations will be logged, but access will not be blocked
* Enforcing -- policy violations will be blocked (and logged).

To check on the SELinux mode, run 

```
$ getenforce
Enforcing
```

## Find the Label of Files and Processes 

Many common commands can be used to view SELinux labels. Try the following:

* `ls -Z`
* `ps -Z`
* `netstat -Z` (you'll probably need to sudo)

Lots of commands have a `-Z`. 

## SELinux Broke My App! 

You're likely here because SELinux isn't letting your web app run. Typically, this takes the form of a mysterious 
`File Not Found` or `Permission Denied` error. 

The good news is that there's a very strong chance that one of two things is wrong:

* a file or folder has the wrong label
* you need to enable an optional part of the policy with a SELinux boolean

### Find the Problem

On a RHEL-based system, you can find the logs from SELinux in `/var/log/audit/audit.log` (you'll probably need to sudo to view them.) That file will likely have SELinux lines that look like: 
```
type=AVC msg=audit(1534218422.277:277403): avc:  denied  { getattr } for  pid=11736 comm="java" path="/srv/oulib/dspace/config/local.cfg" dev="xvda1" ino=377703291 scontext=system_u:system_r:tomcat_t:s0 tcontext=unconfined_u:object_r:var_t:s0 tclass=file
```

Let's break that down:

* `type=AVC` identifies this as an SELinux error
* `msg=audit(1534218422.277:277403):` is mainly important for the timestamp there in the middle
* `avc:  denied  { getattr }` indicates what action was rejected 
* `for  pid=11736 comm="java"` tells us what process tried to act
* `path="/srv/oulib/dspace/config/local.cfg" dev="xvda1" ino=377703291` tells us what resource the proces was acting on.
* `scontext=system_u:system_r:tomcat_t:s0` gives us the label of the process  
* `tcontext=unconfined_u:object_r:var_t:s0 tclass=file` gives us the label of the resource that was accessed

So the TLDR of that is that  a `java` process running  with type `tomcat_t` was not able to access a file whose 
label gives it the type `var_t`.

You can use the tool `audit2why` to get a little bit more information. If we run `cat /var/log/audit/audit.log | auditw2hy` 
we'll see expanded entries that look like this:

```
type=AVC msg=audit(1534218422.277:277403): avc:  denied  { getattr } for  pid=11736 comm="java" path="/srv/oulib/dspace/config/local.cfg" dev="xvda1" ino=377703291 scontext=system_u:system_r:tomcat_t:s0 tcontext=unconfined_u:object_r:var_t:s0 tclass=file

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```

Of course! We're running our app in [`/srv`](https://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/srv.html) but we haven't
adjusted our policy to let SELinux know that  it's OK for `tomcat` to use files in that location. The good news is that the targeted policy already includes the rules we need, we just have to update the labels on the contentsw of `/srv`. 

### Setting the SELinux Label for Files and Folders 

The easiest way to fix that is to copy the configuration from a location like `/var/lib/tomcat/webapps` that comes configured
as part of the tomcat install. We can use `ls -Z` to find the context that we need:

```
$ ls -Z /var/lib/tomcat/webapps/
drwxr-xr-x. tomcat tomcat system_u:object_r:tomcat_var_lib_t:s0 examples
drwxr-xr-x. root   tomcat system_u:object_r:tomcat_var_lib_t:s0 host-manager
drwxr-xr-x. root   tomcat system_u:object_r:tomcat_var_lib_t:s0 manager
drwxr-xr-x. tomcat tomcat system_u:object_r:tomcat_var_lib_t:s0 ROOT
drwxr-xr-x. tomcat tomcat system_u:object_r:tomcat_var_lib_t:s0 sample
```
With that information, we should be able to fix the problem with the following steps. First we update the SELinux policy:
```
semanage fcontext -a -t tomcat_var_lib_t "/srv(/.*)?"
```
and then we apply that policy to the existing files in `/srv`:
```
restorecon -vR /srv/`   
```
and we should be good to go!

### SELinux Booleans

When you run audit2allow, sometimes you'll see a `Was caused by:` message that refers to a boolean, like `The boolean 
nis_enabled was set incorrectly.` 

This generally means that you tried to do something that would be allowed by an optional part of the `targeted` policy, 
but you haven't enabled that option. These options are called SELinux Booleans, and we can use `semanage` to work with them as well.

You can list list all SELinux booleans in the installed policy:
```sudo semanage boolean --list```

or list just the ones that have been  modified:
```sudo semanage boolean --list -C```

To enable a boolean, do:
```
sudo semanage boolean --modify --on httpd_can_network_connect
```
To disabable:
```
sudo semanage boolean --modify --off httpd_can_network_connect_db
```

## Resources for Further Study

The following are great:

* [SELinux Coloring Book](https://github.com/mairin/selinux-coloring-book)
* RedHat Summit  Presentation - SELinux for Mere Mortals [video](https://www.youtube.com/watch?v=_WOKRaM-HI4
 ) and [slides](http://people.redhat.com/tcameron/Summit2018/selinux/SELinux_for_Mere_Mortals_Summit_2018.pdf)
* [RedHat SELinux Admin Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index)
* [Centos SELinux HowTo](https://wiki.centos.org/HowTos/SELinux)
* [Fedora SELinux Pages](https://fedoraproject.org/wiki/SELinux)
