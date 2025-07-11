dnl
dnl
dnl   Authors:
dnl    Lars Fenneberg		<lf@elemental.net>	 
dnl
dnl   This software is Copyright 1996-2000 by the above mentioned author(s), 
dnl   All Rights Reserved.
dnl
dnl   The license which is distributed with this software in the file COPYRIGHT
dnl   applies to this software. If your distribution is missing this file, you
dnl   may request it from <reubenhwk@gmail.com>.
dnl
dnl

dnl If adding rcX to version, be sure to separate with a '-' 
AC_INIT([radvd],[2.20])
AC_CONFIG_SRCDIR(radvd.c)
AC_CANONICAL_TARGET
AM_INIT_AUTOMAKE

m4_ifdef([AM_SILENT_RULES], [AM_SILENT_RULES(yes)])


AC_MSG_CHECKING([architecture])
case "$target" in
	*linux*)
		AC_MSG_RESULT(linux)
		AC_DEFINE([_GNU_SOURCE], [], [whether compiling on Linux, glibc>=2.8 does not expose in6_pktinfo otherwise..])
                arch=linux
        ;;
        *bsd*)
		AC_MSG_RESULT(bsd44)
                arch=bsd44
        ;;
        *darwin*)
		AC_DEFINE([__APPLE_USE_RFC_2292], [1], [OS X 10.9 wont build radvd without this])
		AC_MSG_RESULT([bsd44 (darwin)])
                arch=bsd44
        ;;
        *)      
		AC_MSG_RESULT(unknown: $target)
		AC_MSG_ERROR(
[currently only Linux and BSD 4.4 with NRLs IPv6 code are
supported. If you have such a system and it is not guessed correctly 
you must specify it with --target on the configure command line])
        ;;
esac

dnl Determine CC and preset CFLAGS
AC_PROG_CC_C99
AC_PROG_RANLIB
AM_PROG_AR

AC_ARG_WITH([stack-protector], AS_HELP_STRING([--without-stack-protector], [Build without -fstack-protector]),[],[with_stack_protector=yes])
AM_CONDITIONAL(HAVE_STACK_PROTECTOR, test x"$with_stack_protector" = xyes)

AC_ARG_WITH([check], AS_HELP_STRING([--without-check], [Build without check unit testing framework]),[],[with_check=no])
# AC_ARG_WITH provides HAVE_CHECK*; but that does not say if the library is present
check_version=0.9.4
AS_IF([test "x$with_check" = "xyes"], [
	# only check_all gets linked; NOT radvd / radvdump
	# PKG_CHECK_MODULES's default behavior does not export any variable that
	# says if the module is present or not.  $pkg_cv_CHECK_{CFLAGS,LIBS} are
	# also taken from either env or package, but the package can have them
	# empty, so we must define the true/false.
	# See https://autotools.info/pkgconfig/pkg_check_modules.html
	PKG_CHECK_MODULES([CHECK], [check >= ${check_version}], [found_yes=yes], [found_check=no])
])
AM_CONDITIONAL(HAVE_CHECK, test x"$with_check" = xyes)
if test "x$found_check" = xno && test "x$with_check" = xyes; then
	AC_MSG_ERROR("check >= ${check_version} required but not found")
fi

dnl Determine of netlink is available
AC_MSG_CHECKING(netlink)
AC_COMPILE_IFELSE([AC_LANG_PROGRAM([[
#include <asm/types.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
]], [[
int
main(int argc, char * argv[])
{
        int sock;
        struct sockaddr_nl snl;

        sock = socket(PF_NETLINK, SOCK_RAW, NETLINK_ROUTE);

        snl.nl_family = AF_NETLINK;
        snl.nl_groups = RTMGRP_LINK;

        return 0;
}
]])],[
AC_DEFINE(HAVE_NETLINK, 1, [Linux netlink])
CONDITIONAL_SOURCES=netlink.${OBJEXT}
AC_MSG_RESULT(yes)
],[
AC_MSG_RESULT(no)
])

dnl clock_gettime is in librt for glibc <2.17
AC_SEARCH_LIBS(clock_gettime, rt)

AC_CHECK_FUNCS(strlcpy, found_strlcpy=yes, found_strlcpy=no)
if test "x$found_strlcpy" = xno; then
	dnl check libbsd for strlcpy
	PKG_CHECK_MODULES([BSD], [libbsd >= 0])
	AC_SEARCH_LIBS(strlcpy, bsd,
					[
						found_bsd_strlcpy=yes
						CFLAGS="$BSD_CFLAGS $CFLAGS"
						LIBS="$BSD_LIBS $LIBS"
					],
					found_bsd_strlcpy=no)
fi

if test "x$found_strlcpy" = xno && test "x$found_bsd_strlcpy" = xno; then
	AC_MSG_ERROR(cannot find strlcpy - upgrade your libc or install libbsd)
fi

dnl Needed for normal compile
AC_PROG_INSTALL
AC_PATH_PROG(RM, rm, NOTFOUND)
if test "x$RM" = xNOTFOUND; then
	AC_MSG_ERROR(can not find rm in your path - check PATH)
fi
AC_PATH_PROG(SED, sed, NOTFOUND)
if test "x$SED" = xNOTFOUND; then
	AC_MSG_ERROR(can not find sed in your path - check PATH)
fi
dnl Not needed
AC_PATH_PROG(LN, ln)
AC_PROG_YACC
AM_PROG_LEX(noyywrap)
AC_PATH_PROG(TAR, tar)
AC_PATH_PROG(GZIP, gzip)

dnl Check where to put the logfile
AC_MSG_CHECKING(where to put logfile)
AC_ARG_WITH(logfile,
[  --with-logfile          Path to the radvd logfile [/var/log/radvd.log]],
	PATH_RADVD_LOG=$withval,
	PATH_RADVD_LOG=/var/log/radvd.log)
AC_MSG_RESULT($PATH_RADVD_LOG)

dnl Check where to put the pidfile
AC_MSG_CHECKING(where to put pidfile)
AC_ARG_WITH(pidfile,
[  --with-pidfile          Path to the radvd pidfile [/var/run/radvd.pid]],
	PATH_RADVD_PID=$withval,
	PATH_RADVD_PID=/var/run/radvd.pid)
AC_MSG_RESULT($PATH_RADVD_PID)

dnl Check where to put the configfile
AC_MSG_CHECKING(where to find configfile)
AC_ARG_WITH(configfile,
[  --with-configfile       Path to the radvd config file [SYSCONF/radvd.conf]],
	PATH_RADVD_CONF=$withval,
	[eval PATH_RADVD_CONF=${sysconfdir}/radvd.conf])
PATH_RADVD_CONF=$(echo $PATH_RADVD_CONF | sed 's/NONE//1')
AC_MSG_RESULT($PATH_RADVD_CONF)

dnl Checking which syslog facility to use
AC_MSG_CHECKING(which syslog facility to use)
AC_ARG_WITH(facility,
[  --with-facility         Syslog facility to use when using syslog logging],
	LOG_FACILITY=$withval,
	LOG_FACILITY=LOG_DAEMON)
AC_MSG_RESULT($LOG_FACILITY)

dnl Checks for libraries.

AC_CHECK_LIB(c, inet_ntop,,
 AC_CHECK_LIB(inet6, inet_ntop,
  LIBS="$LIBS -linet6"
  ,
    AC_MSG_ERROR(can not continue without libinet6.a library - check your LDFLAGS)
 )
)
# prevent caching
unset ac_cv_lib_inet6_inet_ntop

dnl Checks for header files.
AC_HEADER_STDC
AC_CHECK_HEADERS( \
	getopt.h \
	ifaddrs.h \
	linux/if_arp.h \
	machine/limits.h \
	machine/param.h \
	net/if_arp.h \
	net/if_dl.h \
	net/if_types.h \
	sys/param.h \
	sys/sockio.h \
	sys/time.h \
	time.h \
)
AC_HEADER_TIME

dnl Checks for typedefs, structures, and compiler characteristics.
AC_MSG_CHECKING(whether struct sockaddr_in6 has sin6_scope_id)
AC_COMPILE_IFELSE(
	[AC_LANG_PROGRAM([[
		#include <stdint.h>
		#include <sys/types.h>
		#include <netinet/in.h>
	]], [[
		static struct sockaddr_in6 ac_sin6;
		uint32_t ac_size = sizeof (ac_sin6.sin6_scope_id);
	]])],
	[
		AC_MSG_RESULT(yes);
		AC_DEFINE(
			[HAVE_SIN6_SCOPE_ID],
			1,
			[whether struct sockaddr_in6 has sin6_scope_id]
			)
	],
	[AC_MSG_RESULT(no)]
)

AC_MSG_CHECKING(whether struct in6_addr has u6_addrXX and defines s6_addrXX)
AC_COMPILE_IFELSE(
	[AC_LANG_PROGRAM(
		[[
			#include <stdint.h>
			#include <netinet/in.h>
		]], [[
			static struct in6_addr in6_u;
			uint16_t u =  in6_u.s6_addr16[0];
		]]
	)],
	[
		AC_MSG_RESULT(yes);
		AC_DEFINE([HAVE_IN6_ADDR_S6_ADDR], 1, [whether struct in6_addr has u6_addrXX and defines s6_addrXX])
	],
	[AC_MSG_RESULT(no)]
)

dnl Checks for library functions.
AC_CHECK_FUNCS(getopt_long)
AC_CHECK_FUNCS(ppoll)
AC_CHECK_FUNCS(sysctl)

CONDITIONAL_SOURCES="device-${arch}.${OBJEXT} ${CONDITIONAL_SOURCES}"
if test x${arch} = xlinux ; then
	CONDITIONAL_SOURCES="privsep-${arch}.${OBJEXT} ${CONDITIONAL_SOURCES}"
	AC_DEFINE(USE_PRIVSEP, 1, [Use privsep])
fi
AC_SUBST(CONDITIONAL_SOURCES)

AC_SUBST(VERSION)
AC_SUBST(PATH_RADVD_CONF)
AC_SUBST(PATH_RADVD_PID)
AC_SUBST(PATH_RADVD_LOG)
AC_SUBST(LOG_FACILITY)

# Default value for sbindir
prefix_temp=$prefix
exec_prefix_temp=$exec_prefix

test "${prefix}" = "NONE" && prefix="${ac_default_prefix}"
test "${exec_prefix}" = "NONE" && exec_prefix='${prefix}'

# Initial Value is $exec_prefix/sbin
sbintemp="${sbindir}"

# Expands to $prefix/sbin
eval sbintemp=\"${sbintemp}\"
# Expands to /usr/local/sbin or /usr/sbin if --prefix is passed
eval sbintemp=\"${sbintemp}\"
SBINDIR=${sbintemp}

AC_SUBST(SBINDIR)

PKG_PROG_PKG_CONFIG
AC_ARG_WITH([systemdsystemunitdir],
     [AS_HELP_STRING([--with-systemdsystemunitdir=DIR], [Directory for systemd service files])],,
     [with_systemdsystemunitdir=auto])
AS_IF([test "x$with_systemdsystemunitdir" = "xyes" -o "x$with_systemdsystemunitdir" = "xauto"], [
     def_systemdsystemunitdir=$($PKG_CONFIG --variable=systemdsystemunitdir systemd)

     AS_IF([test "x$def_systemdsystemunitdir" = "x"],
   [AS_IF([test "x$with_systemdsystemunitdir" = "xyes"],
    [AC_MSG_ERROR([systemd support requested but pkg-config unable to query systemd package])])
    with_systemdsystemunitdir=no],
   [with_systemdsystemunitdir="$def_systemdsystemunitdir"])])
AS_IF([test "x$with_systemdsystemunitdir" != "xno"],
      [AC_SUBST([systemdsystemunitdir], [$with_systemdsystemunitdir])])
AM_CONDITIONAL([HAVE_SYSTEMD], [test "x$with_systemdsystemunitdir" != "xno"])

AM_CONFIG_HEADER(config.h)
AC_CONFIG_FILES([Makefile \
	radvd.service \
	redhat/systemd/radvd.spec \
	redhat/SysV/radvd.spec \
])
AC_OUTPUT

cat << EOF

Your build configuration:

	CPPFLAGS = $CPPFLAGS
	CFLAGS = $CFLAGS
	LDFLAGS = $LDFLAGS
	Arch = ${arch}
	Extras: ${CONDITIONAL_SOURCES}
	prefix: $prefix
	PID file: $PATH_RADVD_PID
	Log file: $PATH_RADVD_LOG
	Config file: $PATH_RADVD_CONF
	Radvd version: $VERSION

EOF

