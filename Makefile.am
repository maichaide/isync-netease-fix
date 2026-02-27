# SPDX-FileCopyrightText: 2000-2002 Michael R. Elkins <me@mutt.org>
# SPDX-FileCopyrightText: 2002-2022 Oswald Buddenhagen <ossi@users.sf.net>
# SPDX-License-Identifier: GPL-2.0-or-later

SUBDIRS = src
bin_SCRIPTS = mbsync-get-cert
EXTRA_DIST = LICENSES VERSION debian isync.spec $(bin_SCRIPTS)

cov-scan: clean
	/opt/cov-analysis-*/bin/cov-build --dir cov-int $(MAKE)
	tar cavf isync-cov.tar.xz cov-int

deb:
	CFLAGS= INSTALL= dpkg-buildpackage -b --no-sign

dist-hook:
	find $(distdir)/debian \( -name .#\*# -o -type l \) -print0 | xargs -0r rm -rf
	-cd $(distdir)/debian && test -f .gitignore && rm -rf `cut -c2- .gitignore` .gitignore

dist-sign: dist
	gpg -b -a $(PACKAGE)-$(VERSION).tar.gz

rpm: dist
	CFLAGS="-O2 -mtune=core2" rpmbuild --clean -ta $(PACKAGE)-$(VERSION).tar.gz

rpm-ia32: dist
	CFLAGS="-O2 -m32 -march=i686" rpmbuild --target i686-unknown-linux --clean -ta $(PACKAGE)-$(VERSION).tar.gz

doc_DATA = README TODO NEWS ChangeLog AUTHORS
