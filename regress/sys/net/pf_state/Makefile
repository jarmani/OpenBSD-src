#	$OpenBSD: Makefile,v 1.8 2019/05/10 02:22:34 bluhm Exp $

# The following ports must be installed:
#
# python-2.7          interpreted object-oriented programming language
# scapy               powerful interactive packet manipulation in python

.include <bsd.own.mk>

.if ! (make(clean) || make(cleandir) || make(obj))
PF_ENABLED !!=	${SUDO} pfctl -si | grep '^Status: Enabled ' || true
PF_ANCHOR !!=	${SUDO} pfctl -sr | grep '^anchor "regress" all$$' || true
.endif

.if ! exists(/usr/local/bin/python2.7) || ! exists(/usr/local/bin/scapy)
.BEGIN:
	@true
regress:
	@echo Install python and the scapy module for additional tests.
	@echo SKIPPED
.endif

.if ! empty(PF_ENABLED) && empty(PF_ANCHOR)
.BEGIN:
	@true
regress:
	@echo This test cannot run with pf filtering packets on localhost.
	@echo Either disable pf or add a line 'anchor "regress"' to pf.conf.
	@echo There the test will add a stateless rule that passes all packets.
	@echo SKIPPED
.endif

# This test needs a manual setup of two machines
# Set up machines: LOCAL REMOTE
# LOCAL is the machine where this makefile is running.
# REMOTE is running OpenBSD with echo server to test PMTU
# FAKE is an non existing machine in a non existing network.
# REMOTE_SSH is the hostname to log in on the REMOTE machine.

# Configure Addresses on the machines.
# Adapt interface and addresse variables to your local setup.
#
LOCAL_IF ?=
REMOTE_SSH ?=

LOCAL_ADDR ?=
REMOTE_ADDR ?=
FAKE_NET ?=
FAKE_NET_ADDR ?=

.if empty (LOCAL_IF) || empty (REMOTE_SSH) || \
    empty (LOCAL_ADDR) || \
    empty (REMOTE_ADDR) || \
    empty (FAKE_NET) || \
    empty (FAKE_NET_ADDR)
.BEGIN:
	@true
regress:
	@echo This tests needs a remote machine to operate on
	@echo LOCAL_IF REMOTE_SSH LOCAL_ADDR REMOTE_ADDR FAKE_NET FAKE_NET_ADDR
	@echo are empty.  Fill out these variables for additional tests.
	@echo SKIPPED
.endif

.MAIN: all

.if make (regress) || make (all)
.BEGIN: pf.conf addr.py
	@echo
	${SUDO} true
	ssh -t ${REMOTE_SSH} ${SUDO} true
	rm -f stamp-pfctl
.endif

# Create python include file containing the addresses.
addr.py: Makefile
	rm -f $@ $@.tmp
	echo 'LOCAL_IF = "${LOCAL_IF}"' >>$@.tmp
.for var in LOCAL REMOTE FAKE_NET
	echo '${var}_ADDR = "${${var}_ADDR}"' >>$@.tmp
.endfor
	echo 'FAKE_NET = "${FAKE_NET}"' >>$@.tmp
	mv $@.tmp $@

# load the pf rules into the kernel of the REMOTE and LOCAL machine
REGRESS_SETUP +=	stamp-pfctl
stamp-pfctl: addr.py pf.conf
	cat addr.py ${.CURDIR}/pf.conf | pfctl -n -f -
	cat addr.py ${.CURDIR}/pf.conf | \
	    ssh ${REMOTE_SSH} ${SUDO} pfctl -a regress -f -
.if ! empty(PF_ANCHOR)
	echo 'pass on ${LOCAL_IF} no state' | ${SUDO} pfctl -a regress -f -
.endif
	@date >$@

# Set variables so that make runs with and without obj directory.
# Only do that if necessary to keep visible output short.
.if ${.CURDIR} == ${.OBJDIR}
PYTHON =	python2.7 -u ./
.else
PYTHON =	PYTHONPATH=${.OBJDIR} python2.7 -u ${.CURDIR}/
.endif

REGRESS_TARGETS +=	run-challenge-ack
run-challenge-ack: challenge_ack.py addr.py
	@echo '\n======== $@ ========'
	${SUDO} ${PYTHON}challenge_ack.py

.if ! empty(PF_ANCHOR)
REGRESS_CLEANUP +=	cleanup
cleanup:
	@echo '\n======== $@ ========'
	${SUDO} pfctl -a regress -Fr
	rm -f stamp-pfctl
.endif

CLEANFILES +=		addr.py *.pyc *.log stamp-*

.PHONY: check-setup check-setup-local check-setup-remote

# Check wether the address, route and remote setup is correct
check-setup: check-setup-local check-setup-remote

check-setup-local:
	@echo '\n======== $@ ========'
	ping -n -c 1 ${LOCAL_ADDR}  # LOCAL_ADDR
	route -n get -inet ${LOCAL_ADDR} | grep -q 'flags: .*LOCAL'  # LOCAL_ADDR
	ping -n -c 1 ${REMOTE_ADDR}  # REMOTE_ADDR
	route -n get -inet ${REMOTE_ADDR} | fgrep -q 'interface: ${LOCAL_IF}'  # REMOTE_ADDR LOCAL_IF
	! ping -n -c 1 -w 1 ${FAKE_NET_ADDR}  # FAKE_NET_ADDR
.for ip in FAKE_NET FAKE_NET_ADDR
	route -n get -inet ${${ip}} | grep -q 'flags: .*BLACKHOLE'  # ${ip}
.endfor

check-setup-remote:
	@echo '\n======== $@ ========'
	ssh ${REMOTE_SSH} ping -n -c 1 ${REMOTE_ADDR}  # REMOTE_ADDR
	ssh ${REMOTE_SSH} route -n get -inet ${REMOTE_ADDR} | grep -q 'flags: .*LOCAL'  # REMOTE_ADDR
	ssh ${REMOTE_SSH} ping -n -c 1 ${LOCAL_ADDR}  # LOCAL_ADDR
.for ip in FAKE_NET FAKE_NET_ADDR
	ssh ${REMOTE_SSH} route -n get -inet ${${ip}} | fgrep -q 'gateway: ${LOCAL_ADDR}'  # ${ip} LOCAL_ADDR
.endfor
	ssh ${REMOTE_SSH} netstat -na -f inet -p tcp | fgrep ' *.7 '
	ssh ${REMOTE_SSH} ${SUDO} pfctl -sr | grep '^anchor "regress" all$$'
	ssh ${REMOTE_SSH} ${SUDO} pfctl -si | grep '^Status: Enabled '

.include <bsd.regress.mk>
