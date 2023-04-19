# OBS Reproducible Builds Demo

This is an example of manually building a package locally so it is identical to the one build on [OBS](https://openbuildservice.org/).

The [nachbau](https://github.com/bmwiedemann/reproducibleopensuse/blob/master/nachbau) script is automation to do the same.

### Checkout a simple package

```
# osc -A https://api.opensuse.org checkout home:doreilly:branches:Base:System/hello && cd $_
A    home:doreilly:branches:Base:System
A    home:doreilly:branches:Base:System/hello
A    home:doreilly:branches:Base:System/hello/hello-2.12-reproducible.patch
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz.sig
A    home:doreilly:branches:Base:System/hello/hello.changes
A    home:doreilly:branches:Base:System/hello/hello.keyring
A    home:doreilly:branches:Base:System/hello/hello.spec
At revision 727bd5135f72df8833b1155317851524.
```

### Update the project metadata
From [openSUSE:Reproducible Builds](https://en.opensuse.org/openSUSE:Reproducible_Builds#With_OBS)

```
# osc meta prjconf
Macros:
%source_date_epoch_from_changelog Y
%clamp_mtime_to_source_date_epoch Y
%use_source_date_epoch_as_buildtime Y
%_buildhost reproducible
:Macros 
```

Rebuild on OBS so the new project metadata takes effect.
```
# osc wipebinaries --all
# osc rebuild 15.5 x86_64
```

### Download the RPMs
```
# osc getbinaries --sources 15.5 x86_64
Creating directory "binaries"
Please install the progressbar module
Processing: _buildenv
Processing: _statistics
Processing: hello-2.12.1-150500.110.2.src.rpm
Processing: hello-2.12.1-150500.110.2.x86_64.rpm
Processing: hello-lang-2.12.1-150500.110.2.noarch.rpm
Processing: rpmlint.log
```

Note: the "Build Date" is the most recent date from the changelog
```
# rpm -q --info binaries/hello-2.12.1-150500.110.2.x86_64.rpm | grep Build
Build Date  : Mon May 30 12:00:00 2022
Build Host  : reproducible

# head -2 hello.changes 
-------------------------------------------------------------------
Mon May 30 10:21:23 UTC 2022 - Andreas Stieger <andreas.stieger@gmx.de>
```

### Rebuild locally
First need to extract some metadata from the downloaded RPM.
```
# rpm -q --info binaries/hello-2.12.1-150500.110.2.x86_64.rpm | grep Release
Release     : 150500.110.2

# rpm -qp --queryformat '%{DISTURL}\n' binaries/hello-2.12.1-150500.110.2.x86_64.rpm
obs://build.opensuse.org/home:doreilly:branches:Base:System/15.5/727bd5135f72df8833b1155317851524-hello
```

To workaround [this bug](https://github.com/rpm-software-management/rpm/issues/2343) we need to rebuild with the same number of CPUs as was used on the server.
```
# osc buildlog 15.5 x86_64 | grep qemu-kvm | sed 's/^.*-smp//'
 8
```

Now do a local build with the same metadata.
```
# osc build \
  --clean \
  --release 150500.110.2 \
  --build-opt=--disturl=obs://build.opensuse.org/home:doreilly:branches:Base:System/15.5/727bd5135f72df8833b1155317851524-hello \
  --vm-type=kvm --vm-memory 2048 --jobs 8 \
  --no-service \
   15.5 x86_64
```

### Compare binaries
The signature from the build server needs to removed first.
```
# rpm --delsign binaries/hello-2.12.1-150500.110.2.x86_64.rpm

# sha256sum binaries/hello-2.12.1-150500.110.2.x86_64.rpm /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  binaries/hello-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
```
