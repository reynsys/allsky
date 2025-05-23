platform = $(shell uname -m)
prefix = 

sysconfdir = ${prefix}/etc
exec_prefix = /usr
bindir = ${exec_prefix}/bin
libexecdir = ${exec_prefix}/libexec/allsky
sharedir = ${exec_prefix}/share/allsky

HOMEDIR = $(shell dirname "$$PWD")
SCRIPTSDIR = ${HOMEDIR}/scripts

.DEFAULT_GOAL := all

ROOTCHECK=$(shell id -u)
ifneq ($(ROOTCHECK),0)
  ifeq ($(PKGBUILD),1)
    ROOTCHECK=0
  endif
endif

ifeq ($(PKGBUILD),)
  PKGBUILD=0
endif


ENVFILE := env.json
UNINSTALLFILES := $(DESTDIR)$(sysconfdir)/logrotate.d/allsky $(DESTDIR)$(sysconfdir)/rsyslog.d/allsky.conf $(DESTDIR)$(sysconfdir)/systemd/system/allsky.service $(DESTDIR)$(sysconfdir)/systemd/system/allskyperiodic.service

%:
	@echo `date +%F\ %R:%S` nothing to do for $@

ifneq ($(ROOTCHECK), 0)
install uninstall:
	@echo This must be run with root permissions.
	@echo Please run \'sudo make $@\'
else # Root check passed

uninstall:
	@echo `date +%F\ %R:%S` Removing installed files: $(UNINSTALLFILES)
	@rm -f $(UNINSTALLFILES)
	@echo `date +%F\ %R:%S` Complete.
	@echo `date +%F\ %R:%S` NOTE: Config files were \-NOT\- removed.
	@echo `date +%F\ %R:%S` To remove config files, please run \'sudo make remove_configs\'
.PHONY : uninstall

ifeq ($(PKGBUILD),1)

remove_configs:
	@echo `date +%F\ %R:%S` Removing config path and files $(DESTDIR)$(sysconfigdir)/allsky
	@rm -rf $(DESTDIR)$(sysconfigdir)/allsky

createDirs:
	@echo `date +%F\ %R:%S` Creating directory structures...
	@if [ ! -e $(DESTDIR)$(sysconfdir)/allsky ]; then mkdir -p $(DESTDIR)$(sysconfdir)/allsky; fi
	@if [ ! -e $(DESTDIR)$(sysconfdir)/profile.d ]; then mkdir -p $(DESTDIR)$(sysconfdir)/profile.d; fi
	@if [ ! -e $(DESTDIR)$(sysconfdir)/systemd/system ]; then mkdir -p $(DESTDIR)$(sysconfdir)/systemd/system; fi
	@if [ ! -e $(DESTDIR)$(sysconfdir)/udev/rules.d ]; then mkdir -p $(DESTDIR)$(sysconfdir)/udev/rules.d; fi
.PHONY : createDirs

$(ENVFILE):
	@if [ ! -e $(DESTDIR)$(sysconfdir)/allsky/$@ ]; then \
	  echo `date +%F\ %R:%S` Copying default $@; \
	  cp $@.repo > env; \
	  install -m 0664 env $(HOMEDIR)/$@; \
	  rm -f env; \
	fi
# TODO: In PKGBUILD mode, where is the "allsky" directory?  $(ENVFILE) needs to go in it.
# Is $(HOMEDIR) the correct location, or should it be $(DESTDIR)$(sysconfdir)/allsky  ?
.PHONY : $(ENVFILE)

install: createDirs $(ENVFILE)
	@echo `date +%F\ %R:%S` Setting up udev rules...
	@install -D -m 0644 asi.rules $(DESTDIR)$(sysconfdir)/udev/rules.d/
	@echo `date +%F\ %R:%S` Setting up logging...
	@install -D -m 0644 allsky.logrotate.repo $(DESTDIR)$(sysconfdir)/logrotate.d/allsky
	@install -D -m 0644 allsky.rsyslog.repo $(DESTDIR)$(sysconfdir)/rsyslog.d/allsky.conf
	@echo `date +%F\ %R:%S` Setting allsky to auto start...
	@sed -e "s|XX_ALLSKY_OWNER_XX|allsky|" -e "s|XX_ALLSKY_HOME_XX|$(bindir)|" allsky.service.repo > allsky.service
	@install -m 0644 allsky.service $(DESTDIR)$(sysconfdir)/systemd/system/
	@rm -f allsky.service
	@echo `date +%F\ %R:%S` Setting up home environment variable...
	@echo "export ALLSKY_TMP=/tmp\nexport ALLSKY_CONFIG=$(DESTDIR)$(sysconfdir)/allsky\nexport ALLSKY_SCRIPTS=$(DESTDIR)$(libexecdir)\nexport ALLSKY_NOTIFICATION_IMAGES=$(DESTDIR)$(sharedir)\nexport ALLSKY_IMAGES=/home/allsky/images/\nPATH=${PATH}:${DESTDIR}$(libexedir)" > $(DESTDIR)$(sysconfdir)/profile.d/allsy.sh


else # Not in package build mode ######################################

remove_configs:
	@echo `date +%F\ %R:%S` Removing config path and files ../config
	@rm -rf ../config
.PHONY : remove_configs

createDirs:
	@echo `date +%F\ %R:%S` Creating directory structures...
	@if [ ! -e ../config ]; then mkdir -p ../config; chown $(SUDO_USER):$(SUDO_USER) ../config; fi
.PHONY : createDirs

install: createDirs
	@echo `date +%F\ %R:%S` Setting up udev rules...
	@install -D -m 0644 asi.rules $(DESTDIR)$(sysconfdir)/udev/rules.d/
	@echo `date +%F\ %R:%S` Setting up logging...
	@install -D -m 0644 allsky.logrotate.repo $(DESTDIR)$(sysconfdir)/logrotate.d/allsky
	@install -D -m 0644 allsky.rsyslog.repo $(DESTDIR)$(sysconfdir)/rsyslog.d/allsky.conf
	@udevadm control -R
	@udevadm trigger
	@echo `date +%F\ %R:%S` Setting allsky to auto start...
	@if [ -e /etc/xdg/lxsession/LXDE-pi/autostart ]; then \
	  sed -i '/allsky.sh/d' /etc/xdg/lxsession/LXDE-pi/autostart; fi
	@# Don't use the /lib/systemd/system directory
	@sed -e "s|XX_ALLSKY_OWNER_XX|$(SUDO_USER)|" -e "s|XX_ALLSKY_HOME_XX|$(HOMEDIR)|" -e "s|XX_ALLSKY_SCRIPTS_XX|$(SCRIPTSDIR)|" allsky.service.repo > allsky.service
	@install -m 0644 allsky.service $(DESTDIR)$(sysconfdir)/systemd/system/
	@rm -f allsky.service
	@if [ -e /lib/systemd/system/allsky.service ]; then rm -f /lib/systemd/system/allsky.service; fi
	@sed -e "s|XX_ALLSKY_OWNER_XX|$(SUDO_USER)|" -e "s|XX_ALLSKY_SCRIPTS_XX|$(SCRIPTSDIR)|" allskyperiodic.service.repo > allskyperiodic.service
	@install -m 0644 allskyperiodic.service $(DESTDIR)$(sysconfdir)/systemd/system/
	@rm -f allskyperiodic.service
	@if [ -e /lib/systemd/system/allskyperiodic.service ]; then rm -f /lib/systemd/system/allskyperiodic.service; fi
	@systemctl daemon-reload
	@systemctl enable allsky
	@systemctl enable allskyperiodic
	@echo `date +%F\ %R:%S` Setting up ALLSKY_HOME environment variable...
	@echo "export ALLSKY_HOME=$(HOMEDIR)\nPATH=${PATH}:$(SCRIPTSDIR)" > $(DESTDIR)$(sysconfdir)/profile.d/allsky.sh
.PHONY : install
endif # Package build check
endif # Root check
