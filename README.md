# OBS Reproducible Builds Demo

This is an example of manually building a package locally so it is identical to the one build on [OBS](https://openbuildservice.org/).

The [nachbau](https://github.com/bmwiedemann/reproducibleopensuse/blob/master/nachbau) script is automation to do the same.

### Checkout a simple package

```
> osc bco openSUSE:Factory/hello

Note: The branch has been created of a different project,
              Base:System,
      which is the primary location of where development for
      that package takes place.
      That's also where you would normally make changes against.
      A direct branch of the specified package can be forced
      with the --nodevelproject option.

A    home:doreilly:branches:Base:System
A    home:doreilly:branches:Base:System/hello
A    home:doreilly:branches:Base:System/hello/hello-2.12-reproducible.patch
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz
A    home:doreilly:branches:Base:System/hello/hello-2.12.1.tar.gz.sig
A    home:doreilly:branches:Base:System/hello/hello.changes
A    home:doreilly:branches:Base:System/hello/hello.keyring
A    home:doreilly:branches:Base:System/hello/hello.spec
At revision 727bd5135f72df8833b1155317851524.

> cd home:doreilly:branches:Base:System/hello
```

### Update the project metadata
From [openSUSE:Reproducible Builds](https://en.opensuse.org/openSUSE:Reproducible_Builds#With_OBS)

```
> osc meta prjconf -F - <<EOF
Macros:
%source_date_epoch_from_changelog Y
%clamp_mtime_to_source_date_epoch Y
%use_source_date_epoch_as_buildtime Y
%_buildhost reproducible
:Macros 
EOF
```

Rebuild on OBS so the new project metadata takes effect.
```
> osc wipebinaries --all
> osc rebuild
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
```

If you get questions like the following you can answer with always trust.
```
The build root needs packages from project 'openSUSE:Tumbleweed'.
Note that malicious packages can compromise the build result or even your system.
Would you like to ...
0 - quit (default)
1 - always trust packages from 'openSUSE:Tumbleweed'
2 - trust packages just this time
? 1
adding 'openSUSE:Tumbleweed' to oscrc: ['https://api.opensuse.org']['trusted_prj']
```

### Compare binaries
Make sure your rpm version is at least 4.18, in rpm 4.14.3 from Leap 15.5 there is a bug in delsign.
The signature from the build server needs to removed first. Then you can compare, e.g. visually by the hash or with diff.
```
> rpm --version
RPM version 4.18.0

> rpm --delsign binaries/*.rpm

> sha256sum binaries/*.rpm /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/{SRPMS,RPMS/{x86_64,noarch}}/*.rpm | rev | sort | rev
52868402cca5d79c06ba972207d7d6eb95634dd3b6d44d231913b81f21f501d9  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
52868402cca5d79c06ba972207d7d6eb95634dd3b6d44d231913b81f21f501d9  binaries/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm
f32a1ae72b9630fd08aa59e47682b96485a7a15d7a0a27900678be071f422607  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
f32a1ae72b9630fd08aa59e47682b96485a7a15d7a0a27900678be071f422607  binaries/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
90d7779f5dcc757f5afe6ec317a0915787a1a071f92bc9ced83da86ce7174566  binaries/hello-2.12.1-150500.110.2.x86_64.rpm
1aa7937a2053ebece6d2eb7989785e8faafc6ac30e551caa8af089d8d5e125cc  binaries/hello-2.12.1-150500.110.2.src.rpm
1aa7937a2053ebece6d2eb7989785e8faafc6ac30e551caa8af089d8d5e125cc  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/SRPMS/hello-2.12.1-150500.110.2.src.rpm
ae8d2ce8ea5fac6e30d32102a8be8440555920b599b07b63f920fcede630b112  /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/noarch/hello-lang-2.12.1-150500.110.2.noarch.rpm
ae8d2ce8ea5fac6e30d32102a8be8440555920b599b07b63f920fcede630b112  binaries/hello-lang-2.12.1-150500.110.2.noarch.rpm

> diff -rs binaries /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64
Only in binaries: _buildenv
Only in binaries: hello-2.12.1-150500.110.2.src.rpm
Binary files binaries/hello-2.12.1-150500.110.2.x86_64.rpm and /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm are identical
Binary files binaries/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm and /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debuginfo-2.12.1-150500.110.2.x86_64.rpm are identical
Binary files binaries/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm and /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-debugsource-2.12.1-150500.110.2.x86_64.rpm are identical
Only in binaries: hello-lang-2.12.1-150500.110.2.noarch.rpm
Only in binaries: rpmlint.log
Only in binaries: _statistics
> diff -rs binaries /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/noarch
> diff -rs binaries /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/SRPMS
```

If there is a difference you can use diffoscope to make it easier to read, as it examines the content of archives and parses binary fromats.
```
> diffoscope binaries/hello-2.12.1-150500.110.2.x86_64.rpm /var/tmp/build-root/15.5-x86_64/.mount/.build.packages/RPMS/x86_64/hello-2.12.1-150500.110.2.x86_64.rpm
```
