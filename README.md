# Finding adequate buildrequires in order to minimize product sizes

`Version 0.1, 2023-02-16`

This document tries to summaries a discussion with:
Michael Schroeder <mls@suse.com>
Ludwig Nussel <lnussel@suse.com>
Alexander Herzig <aherzig@suse.com>
Jiri Srain <jsrain@suse.com>
Frederic Crozat <fcrozat@suse.com>
Dominique Leuenberger <dleuenberger@suse.com>
Fabian Vogt <fvogt@suse.de>
Thorsten Kukuk <kukuk@suse.com>
Stefan Schubert <schubi@suse.com>

## The Problem

Product like ALP are still too big. The reason for it is mainly the fact that programs are still linked
against libraries which will be NOT used in the regarding product at all. Even worse, if we do not support these
libraries at all in these products.

As example, on ALP we use and support only SELinux, but many tools, especially systemd, are still linked
against SELinux and apparmor.

Or for SLE Micro we don't need and support kerberos, but many applications or libraries are linked
against it. As result, since the customers have some non-functional kerberos packages, we need to provide
maintenance updates and the customer needs to reboot for nothing. Or gets confused since there is AppArmor
which we don't use.

## What we do not want..

... is having product specific macros for this, like we have with is_opensuse. This will only lead to
constructs like:

```bash
%if (is_opensuse || is_ALP) && !is_automotive
```

## The Idea Is...

... to enable/disable BuildRequires for packages per Project:
e.g. 
- systemd is build with everything in Factory, but without AppArmor in ALP
- curl is build with everyhing in Factory, but without Kerberos in SLE  Micro
- All packages are build without documentation in Automotive

So a spec files should e.g. look like:

```bash
%if {with ldap}
BuildRequires openldap-devel
%endif
%if {with apparmor}
BuildRequires apparmor-devel
%endif
```

The definition of these flags will be done in the OBS prjconf settings of each
project via bcond settings:

https://openbuildservice.org/help/manuals/obs-user-guide/cha.obs.prjconfig.html
https://en.opensuse.org/openSUSE:Build_Service_prjconf#%25bcond

## Is bcond The Right Way ?

bcond has some "obscure" behaviour regarding default values, when they have been set
automatically and how they can reset again. E.G.:
The issue used to be that the macros behind the bcond used defined() instead of a check
for eg 1/0. So once you turned something on or off via prjconf it wasn't possible to flip
the switch again in a derived project. So that is kind of inconvenient should we ever
have the need to change our minds e.g. in a service pack or when someone needs to change
the feature set in eg a devel project.

Supposed there will be OBS projects that e.g. build some container that needs e.g. curl
with kerberos enabled. That project would build against the ALP main project and link
curl from ALP there to rebuild with the kerberos bcond enabled.
Now if the ALP projconf already disabled kerberos by defining %_without_kerberos then
the container project can't enable it again as it's not possible to undefine macros that way.

Perhaps a solution for this issue could be to restrict the usage of bcond in our OBS.
Have a look to :
https://github.com/rpm-software-management/rpm/blob/34c2ba3c6a80a778cdf2e42a9193b3264e08e1b3/macros.in#L100-L141
The author suggests never use without_foo, _with_foo, _without_foo but only with_foo.
And perhaps we could check these restrictions via rpmlint ?
We need some RPM/OBS experts here ;-)

## How To Go On ?

### First Step

- Finding out if settings with bcond is the right way. Comments from mls and Dominique
  would be very helpful. ;-)
- Finding out which packages in factory are already using such kind of general "tags" in
  the spec files and how they are called. Should we use a general prefix for these tags ?

### Second Step

- Evaluating all packages which are building with apparmor, kerberos, selinux.
  These three tags a real usecases and should be handled at first.
- Try to build these packages without the regarding libraries and check them if they are still
  running correctly.
- Set the bcond flag in the spec files which triggers the build with or without these dependencies.
  The default value will be "with".

