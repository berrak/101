## Debanize 101

### Use dh-make-golang
Set environment
```bash
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export PATH=$GOBIN
```
Use option type *program|library* since dh-make-golang can easily be confused by the files in the repository.
```bash
mkdir -p ~/PKGDEB/101
cd ~/PKGDEB/101
dh-make-golang -type program github.com/berrak/101
```
Directory and files after above run:
```bash
101
101_0.0~git20180301.36e0591.orig.tar.xz
itp-101.txt
```
Binary application name is not changed by Debian, thus it remains as *101*.
Ignore the text file, its only for Debian internal usage. Lets look into **101** directory.
It contains three git branches:
```bash
git vbr
* master
  pristine-tar
  upstream
```
Switch to pristine-tar branch and confirm have same base name as the original tarball:
```bash
git sw2 pristine-tar
ls -l
101_0.0~git20180301.36e0591.orig.tar.xz.delta
101_0.0~git20180301.36e0591.orig.tar.xz.id
git sw2 master
```
Edit **changelog** in the debian directory.
```bash
cd debian
vi changelog
```
Replace *UNRELEASED* with stable and remove TODO with (Closes: #123456) or whatever.
Next view the file **control**, but initially it is acceptable for the first packaging attempt.
Edit **rules** and add DH_GOPKG stanza as below:
```bash
#!/usr/bin/make -f

export DH_GOPKG := github.com/berrak/101

%:
        dh $@ --buildsystem=golang --with=golang    
```
Now we will try to build it first with fakeroot to identify problems with these files or the repository:
```bash
cd ..
fakeroot debian/rules build
dh build --buildsystem=golang --with=golang
   dh_testdir -O--buildsystem=golang
   dh_update_autotools_config -O--buildsystem=golang
   dh_autoreconf -O--buildsystem=golang
   dh_auto_configure -O--buildsystem=golang
   dh_auto_build -O--buildsystem=golang
	go install -v -p 1 github.com/berrak/101
github.com/berrak/101
   dh_auto_test -O--buildsystem=golang
	go test -v -p 1 github.com/berrak/101
=== RUN   TestMain
--- PASS: TestMain (0.00s)
=== RUN   TestReverse
--- PASS: TestReverse (0.00s)
PASS
ok  	github.com/berrak/101	0.001s
   create-stamp debian/debhelper-build-stamp
```
Nice, it looks promising. Try to build the binary.
```bash
fakeroot debian/rules binary
dh binary --buildsystem=golang --with=golang
   dh_testroot -O--buildsystem=golang
   dh_prep -O--buildsystem=golang
   dh_auto_install -O--buildsystem=golang
	mkdir -p /home/bekr/PKGDEB/101/101/debian/101/usr/share/gocode/src/github.com/berrak/101
	cp -r -T src/github.com/berrak/101 /home/bekr/PKGDEB/101/101/debian/101/usr/share/gocode/src/github.com/berrak/101
   dh_installdocs -O--buildsystem=golang
   dh_installchangelogs -O--buildsystem=golang
   dh_perl -O--buildsystem=golang
   dh_link -O--buildsystem=golang
   dh_strip_nondeterminism -O--buildsystem=golang
   dh_compress -O--buildsystem=golang
   dh_fixperms -O--buildsystem=golang
   dh_strip -O--buildsystem=golang
   dh_makeshlibs -O--buildsystem=golang
   dh_shlibdeps -O--buildsystem=golang
   dh_installdeb -O--buildsystem=golang
   dh_golang -O--buildsystem=golang
   dh_gencontrol -O--buildsystem=golang
dpkg-gencontrol: warning: Depends field of package 101: unknown substitution variable ${shlibs:Depends}
   dh_md5sums -O--buildsystem=golang
   dh_builddeb -u-Zxz -O--buildsystem=golang
dpkg-deb: building package '101' in '../101_0.0~git20180301.36e0591-1_amd64.deb'.
```
We have our debian package built, albeit with a warning about ${shlibs:Depends} from the control file.
We will ignore that for now. Next step is to debanize in a chrooted environment. Set it up with:
```bash
DIST=stretch ARCH=amd64 git-pbuilder create
sudo ln -s /var/cache/pbuilder/base-stretch-amd64.cow base.cow
```
Remove the the new files that above runs have created. Add the debian files to git:
```bash
git add .
git com -m 'Initial packaging'
```
Before we can build in the chroot we have to update the **rules**.
The binaries is below /opt/ZUL after our enterprise name **ZUL**:
```bash
#!/usr/bin/make -f

CURDIR := $(shell pwd)
TMP  = $(CURDIR)/debian/tmp

export DH_GOPKG := github.com/berrak/101

%:
        dh $@ --buildsystem=golang --with=golang
        
override_dh_auto_build:
        go build 101.go

override_dh_auto_install:
        mkdir -p $(TMP)/opt/ZUL/bin
        cp 101 $(TMP)/opt/ZUL/bin        
```
Add changes to git.
Now we can debanize 101 application in the chrooted stretch environment:
```bash
gbp buildpackage --git-pbuilder --git-compression=xz
```

