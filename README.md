# SELinux for Web Nerds in a Hurry

This document is for web developers who need to quickly develop a working relationship with SELinux on RHEL or CentOS without being spending the time to dive very far in to the details. If you have more than a few minutes to spare, I recommend that you take a look at the [resources](#resources) section for deeper, better, more expert guidance.

## Learning Objectives

* Scratch the surface of the core SELinux concepts:  MAC, contexts/labels, RHEL's targeted policy
* Learn the most common cause of SELinux errors and their symptoms
* Learn a few some tools for identifying and fixing those common errors
* Learn about Ansible features related to SELinux

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

## SELinux Broke My App 

You're likely here because SELinux is breaking your web app. Typically, this takes the form of a mysterious 
`File Not Found` or `Permission Denied` error. 

The good news is that there's a strong chance that one of two things is wrong: 
* a file or folder has the wrong label
* you need to enable an optional part of the policy with a SELinux boolean


### Find the Problem

Identifying SELinux Problems - audit2why

We can mostly find what we need to fix in

/var/log/audit/audit.log using audit2why


### Setting the SELinux Label for Files and Folders 

Managing SELinux Context for Application Files


`semanage fcontext -a -t tomcat_var_lib_t "/srv(/.*)?"`

`semanage fcontext -a -e /var/lib/tomcat/`

`restorecon -vR /srv/`   

### Setting SELinux Booleans

Used for things that services could, but don't always, need.


```sudo semanage boolean --list```
```sudo semanage boolean --list -C```


```sudo semanage boolean --modify --on httpd_can_network_connect```
```sudo semanage boolean --modify --on httpd_can_network_connect_db```



## SELinux and Ansible

Ansible has some built in commands, and you can find examples in several of our roles.

https://docs.ansible.com/ansible/2.6/modules/seboolean_module.html
https://docs.ansible.com/ansible/2.6/modules/sefcontext_module.html

"The sefcontext module does not modify existing files to the new SELinux context(s), so it is advisable to first create the SELinux file contexts before creating files, or run restorecon manually for the existing files that require the new SELinux file contexts"




## Resources for Further Study

The following are great:

* [SELinux Coloring Book](https://github.com/mairin/selinux-coloring-book)
* RedHat Summit  Presentation - SELinux for Mere Mortals [video](https://www.youtube.com/watch?v=_WOKRaM-HI4
 ) and [slides](http://people.redhat.com/tcameron/Summit2018/selinux/SELinux_for_Mere_Mortals_Summit_2018.pdf)
* [RedHat SELinux Admin Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index)
* [Centos SELinux HowTo](https://wiki.centos.org/HowTos/SELinux)
* [Fedora SELinux Pages](https://fedoraproject.org/wiki/SELinux)




