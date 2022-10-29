---
title: "Installing Python2 Applications in Recent Ubuntu Releases"
excerpt: "Getting fwbackups (a python2 application) to install on recent Ubuntu releases (since 20.04) which now support python3 as default"
layout: single
classes: wide
categories:
  - Programming
tags:
  - Python
  - Python2
  - Ubuntu
  - fwbackups
---
On Ubuntu 20.04 the support for Python2 is very limited, as Ubuntu moves to [Python3 by default](https://wiki.ubuntu.com/FocalFossa/ReleaseNotes#Python3_by_default).

> ### Python3 by default
> In 20.04 LTS, the python included in the base system is Python 3.8. Python 2.7 has been moved to universe and is not included by default in any new installs.
>
> Remaining packages in Ubuntu which require Python 2.7 have been updated to use /usr/bin/python2 as their interpreter, and /usr/bin/python is not present by default on any new installs. On systems upgraded from previous releases, /usr/bin/python will continue to point to python2 for compatibility. Users who require /usr/bin/python for compatibility on newly-installed systems are encouraged to install the python-is-python3 package, for a /usr/bin/python pointing to python3 instead.

On face value it seems you can just install python2 from the universe and continue on using your python2 applications by pointing to python2 instead of python3, but in reality it proved unworkable. Trying to install python2 dependencies broke the Ubuntu package manager, and they would not install python2-* packages alongside python3 equivalents. This meant if you wanted to still use python2, you would have a broken python3 system that didn't work. This was unacceptable to me as needed a working python3 environment, but relied on python2 packages that were not being migrated to python3.

One such package was [fwbackups](https://github.com/stewartadam/fwbackups), a neat little backup utility which was unfortunately not being migrated to python3. Converting Qt applications was non-trivial for python2->3 migration, and not something I had time to learn how to do, as you essentially had to rewrite all the Qt components.

There were several blogs, or comments written on how to get fwbackups "working", but none of them actually solved the problem of installing fwbackups and keeping a working python3 environment.

> <https://medium.com/@FrancoR/pygtk-for-python-2-7-on-ubuntu-20-04-4e679fefa071>
> <https://linuxize.com/post/how-to-install-pip-on-ubuntu-20.04/#installing-pip-for-python-2>
> <https://github.com/stewartadam/fwbackups/issues/13#issuecomment-1221654243>

The problems with these methods is that when you install the python2-* dependencies, such as `python-gtk2 python-glade2`, if you had the equivalent python3 packages, it would ask to uninstall those first. But I needed those for other applications. This was really the fault of the Ubuntu package manager, as python2 and python3 can peacefully coexist of the same system, in well separated install directories. A more detailed explanation about this I left in the comment in the [PR above](https://github.com/stewartadam/fwbackups/issues/13#issuecomment-1221902187).

My [first attempt](https://github.com/neilsummers/fwbackups-docker) at trying to solve this was to create a docker environment based on the last Ubuntu version that fully supported python2, which was 18.04. While I could get fwbackups working within a docker container, there were some issues which I could not solve, related to the interaction of the docker container and the files needed on the OS outside the docker container, which lead to some security concerns I was not prepared to live with. Also had issues with modifying cron jobs from within the container that needed to run on the main OS. In the end I abandoned the approach.

After some more investigation, I ended up on a new approach, where I hack the package manager to allow both python2 and python3 version of gtk and other dependencies. I documented the approach in a [PR](https://github.com/stewartadam/fwbackups/pull/14) on the original GitHub repo, and provided a script that would fully automate the process. The install script performed a few different tasks which I will break down and explain here.

First I check for the command line option `--prefix` so that a user defined prefix could be used, as I often install manually obtained packages in `/usr/local` instead of `/usr`.
```bash
PREFIX=/usr/local
# https://stackoverflow.com/questions/192249/how-do-i-parse-command-line-arguments-in-bash
for i in "$@"; do
  case $i in
    --prefix=*)
      PREFIX="${i#*=}"
      shift # past argument=value
      ;;
    -*|--*)
      echo "Unknown option $i"
      exit 1
      ;;
    *)
      echo "Unknown argument $i"
      exit 1
      ;;
  esac
done
PATH=${PREFIX}/bin:${PATH}
```

Then update the sources list for Ubuntu Focal 20.04 to include the Bionic 18.04 release, and update apt to update the sources list.
```bash
# add bionic sources to apt (will remove after install)
release=bionic; cat > "/etc/apt/sources.list.d/$release.list"<<EOF
deb http://archive.ubuntu.com/ubuntu $release universe
deb http://archive.ubuntu.com/ubuntu $release multiverse
deb http://security.ubuntu.com/ubuntu $release-security main
EOF
apt update
```

Then we can install the dependencies which do not require the package manager hacks. The first set are dev tools needed and the main python2.7 package, and the second set are the dependencies needed for fwbackups.
```bash
apt install autotools-dev x11-apps tzdata build-essential
apt install cron gettext intltool
apt install python2.7

# can install this list of packages without hacking depends
apt install libatk-adaptor libgtk2.0-0 libglade2-0 libglib2.0-0 python-cairo python-gobject-2 python-cryptography python-enum34 python-crypto
```

Now we install the fwbackups dependencies that previously broke the python3 equivalents. I basically loop through the dependencies, download the package manually, and then install by forcing the dependencies blockers to be warnings, with the `--force-depends` argument. I then have a script that edits the 
```bash
# manually obtained dependency list excluding python-is-python2,python,python:any needed to install packages list below
declare -a packages=( "python-paramiko" "python-gtk2" "python-glade2" "python-notify" )
for p in "${packages[@]}" ; do
  echo "Installing $p and fixing dependencies"
  apt download $p
  dpkg --force-depends -i "$p"*.deb
  rm "$p"*.deb
  # edit /var/lib/dpkg/status to remove Depends line from package
  ./edit_dpkg_status $p
  diff /var/lib/dpkg/status status_new || true   # exit 1 if different which they are
  mv status_new /var/lib/dpkg/status
done
```
The `edit_dpkg_status` script automatically changed the dependencies downloaded by dpkg and removed the dependencies on `python`, `python:any` and `python-is-python2`, which allowed them to now coexist alongside python3 packages, without forcing the python3 packages to be removed (as they required `python-is-python3`, which could not exist with `python-is-python2`).
```bash
#!/usr/bin/python3

import os
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('packages', type=str, nargs='*',
                    help='space separated list of python packages to remove dependencies in /var/lib/dpkg/status')
parser.add_argument('--debug', action='store_true', default=False)
args = parser.parse_args()

exclude_depends = ['python', 'python:any', 'python-is-python2']

def remove_specific_depends(depends_line):
    dependencies = depends_line.replace('Depends: ', '').replace('\n', '').split(',')
    new_dependencies = [d for d in dependencies if d.split()[0] not in exclude_depends]
    if new_dependencies:
        return 'Depends: ' + ', '.join(new_dependencies) + '\n'
    else:
        return None

with open('/var/lib/dpkg/status' if not args.debug else 'status', 'r') as f:
    with open('status_new', 'w') as f_new:
        remove_depends = False
        for line in f:
            if line.startswith('Package:'):
                # start new package block
                package = line.split()[1]
                remove_depends = package in args.packages
            if line.startswith('Depends') and remove_depends:
                new_line = remove_specific_depends(line)
                if new_line is not None:
                    f_new.write(new_line)
            else:
                f_new.write(line)
```

Then I just update the application to point to `python2` directly, instead of `python`, and everything works as expected, and `python3` can continue to function normally.
```bash
#!/usr/bin/python	->	#!/usr/bin/python2
```
Some care when updating the distribution will probably have to happen to make sure these manual edits to the package dependencies does not get overwritten. I will be doing this soon and probably update the PR for Ubuntu 22.04.
