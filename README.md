# OBS Reproducible Builds Demo

This is an example of manually building a package locally so it is identical to the one built on [OBS](https://openbuildservice.org/).

The [nachbau](https://github.com/bmwiedemann/reproducibleopensuse/blob/master/nachbau) script is automation to do the same.

### Checkout a simple package

```
> osc -A https://api.opensuse.org checkout home:doreilly:branches:Base:System/hello && cd $_
A    home:doreilly:branches:Base:System/hello
A    home:doreilly:branches:Base:System/hello/hello-2.12-reproducible.patch
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz.sig
A    home:doreilly:branches:Base:System/hello/hello.changes
A    home:doreilly:branches:Base:System/hello/hello.keyring
A    home:doreilly:branches:Base:System/hello/hello.spec
At revision 727bd5135f72df8833b1155317851524.
```


### Project metadata
The project's metadata has the settings required for [openSUSE:Reproducible Builds](https://en.opensuse.org/openSUSE:Reproducible_Builds#With_OBS)
```
> osc meta prjconf
Macros:
%source_date_epoch_from_changelog Y
%clamp_mtime_to_source_date_epoch Y
%use_source_date_epoch_as_buildtime Y
%_buildhost reproducible
:Macros
```

### Download the RPMs
```
> osc getbinaries --sources --debuginfo 15.5 x86_64
Creating directory "binaries"
Please install the progressbar module
Processing: _buildenv
Processing: _statistics
Processing: hello-2.12.1-150500.110.2.src.rpm
Processing: hello-2.12.1-150500.110.2.x86_64.rpm
Processing: hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
Processing: hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
Processing: hello-lang-2.12.1-150500.110.2.noarch.rpm
Processing: rpmlint.log
```

Note: the "Build Date" is the most recent date from the changelog
```
> rpm -q --info binaries/hello-2.12.1-150500.110.2.x86_64.rpm | grep Build
Build Date  : Mon May 30 12:00:00 2022
Build Host  : reproducible

> head -2 hello.changes
-------------------------------------------------------------------
Mon May 30 10:21:23 UTC 2022 - Andreas Stieger <andreas.stieger@gmx.de>
```

The build log from the server shows the actual build host and build date.
```
> osc buildlog 15.5 x86_64 | grep "started.*at" | head -1
[    1s] lamb15 started "build hello.spec" at Fri Apr 14 12:39:19 UTC 2023.
```

### Rebuild locally
First need to extract some metadata from the downloaded RPM used later with the argument `--release` and `--build-opt=--disturl=` on osc build.
```
> rpm -q --info binaries/hello-2.12.1-150500.110.2.x86_64.rpm | grep Release
Release     : 150500.110.2

> rpm -qp --queryformat '%{DISTURL}\n' binaries/hello-2.12.1-150500.110.2.x86_64.rpm
obs://build.opensuse.org/home:doreilly:branches:Base:System/15.5/727bd5135f72df8833b1155317851524-hello
```

To workaround [this bug](https://github.com/rpm-software-management/rpm/issues/2343) we need to rebuild with the same number of CPUs (argument `--jobs` on osc build) as was used on the server.
```
> osc buildlog 15.5 x86_64 | grep qemu-kvm | sed 's/^.*-smp//'
 8
```

Now do a local build with the same metadata, replace the values with the ones obtained above.
```
> osc build \
  --clean \
  --release 150500.110.2 \
  --build-opt=--disturl=obs://build.opensuse.org/home:doreilly:branches:Base:System/15.5/727bd5135f72df8833b1155317851524-hello \
  --vm-type=kvm --vm-memory 2048 --jobs 8 \
  --no-service \
   15.5 x86_64

Building hello.spec for 15.5/x86_64
Getting buildconfig from server and store to /root/home:doreilly:branches:Base:System/hello/.osc/_buildconfig-15.5-x86_64
Getting buildinfo from server and store to /root/home:doreilly:branches:Base:System/hello/.osc/_buildinfo-15.5-x86_64.xml
Updating cache of required packages
0.0% cache miss. 135/135 dependencies cached.

Skipping verification of package signatures due to secure VM build
Writing build configuration
Running build
VM_ROOT: /var/tmp/build-root/15.5-x86_64/img, VM_SWAP: /var/tmp/build-root/15.5-x86_64/swap
Creating ext3 filesystem on /var/tmp/build-root/15.5-x86_64/img
logging output to /var/tmp/build-root/15.5-x86_64/.build.log...
[    0s] Using BUILD_ROOT=/var/tmp/build-root/15.5-x86_64/.mount
[    0s] Using BUILD_ARCH=x86_64:i686:i586:i486:i386
[    0s] Doing kvm build in /var/tmp/build-root/15.5-x86_64/img
[    0s] 
[    0s] 
[    0s] tumble1 started "build hello.spec" at Fri Jun  2 11:52:34 UTC 2023.

< omitting long build output ...>

[   41s] tumble1 finished "build hello.spec" at Fri Jun  2 11:53:15 UTC 2023.
[   41s] 
[   41s] [   29.591434][    T1] sysrq: Power Off
[   41s] [   29.595561][    T7] reboot: Power down
[   41s] build: extracting built packages...
[   41s] RPMS/noarch/hello-lang-2.12.1-150500.110.2.noarch.rpm
[   41s] RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
[   41s] RPMS/x86_64/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
[   41s] RPMS/x86_64/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
[   41s] SRPMS/hello-2.12.1-150500.110.2.src.rpm
[   41s] OTHER/rpmlint.log
[   41s] OTHER/_statistics

/var/tmp/build-root/15.5-x86_64/.mount/.build.packages/SRPMS/hello-2.12.1-150500.110.2.src.rpm

/var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/noarch/hello-lang-2.12.1-150500.110.2.noarch.rpm
/var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
/var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
/var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm

```

### Compare binaries
Make sure your rpm version is at least 4.18, in rpm 4.14.3 from Leap 15.5 there is a bug in delsign.
The signature from the build server needs to removed first. Then you can compare, e.g. visually by the hash or with diff.
```
> rpm --version
RPM version 4.18.0

> rpm --delsign binaries/*.rpm

> sha256sum binaries/*.rpm /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/{SRPMS,RPMS/{x86_64,noarch}}/*.rpm | rev | sort | rev
58d287358c6c0b5295160cad8943d512227802c998ab88dac7d676b99f92f8fb  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
58d287358c6c0b5295160cad8943d512227802c998ab88dac7d676b99f92f8fb  binaries/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
f6d303f83395fb0e5f353f073665ba4f6ff1cc1d9ff646471481a35edd556625  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
f6d303f83395fb0e5f353f073665ba4f6ff1cc1d9ff646471481a35edd556625  binaries/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  binaries/hello-2.12.1-150500.110.2.x86_64.rpm
c124cd06093d51550c2350de262d11ef0f33802a7ead73e624131af69b707cee  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/SRPMS/hello-2.12.1-150500.110.2.src.rpm
c124cd06093d51550c2350de262d11ef0f33802a7ead73e624131af69b707cee  binaries/hello-2.12.1-150500.110.2.src.rpm
ecf900ca06437116353eb5768fbece960e7506594f76cfc329f9f4a5185afb68  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/noarch/hello-lang-2.12.1-150500.110.2.noarch.rpm
ecf900ca06437116353eb5768fbece960e7506594f76cfc329f9f4a5185afb68  binaries/hello-lang-2.12.1-150500.110.2.noarch.rpm

```

If there is a difference you can use [diffoscope](https://diffoscope.org) to make it easier to read, as it examines the content of archives and parses binary fromats.
```
> diffoscope binaries/hello-2.12.1-150500.110.2.x86_64.rpm /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm

```
