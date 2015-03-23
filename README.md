Introduction
============

This subordinate charm is used to configure nrpe (Nagios Remote Plugin
Executor). It can be related to the nagios charm via the monitors relation and
will pass a monitors yaml to nagios informing it of what checks to monitor.

Principle Relations
===================

This charm can be attatched to any principle charm (via the juju-info relation)
regardless of whether it has implemented the local-monitors or
nrpe-external-master relations. For example,

juju deploy ubuntu
juju deploy nrpe
juju deploy nagios
juju add-relation ubuntu nrpe
juju add-relation nrpe:monitors nagios:monitors

If joined via the juju-info relation the default checks are configured and
additional checks can be added via the monitors config option (see below).

The local-monitors relations allows the principle to request checks to be setup
by passing a monitors yaml and listing them in the 'local' section. It can
also list checks that is has configured by listing them in the remote nrpe
section and finally it can request external monitors are setup by using one of
the other remote types. See "Monitors yaml" below.

Check sources
=============

Check definitions can come from three places:

Default Checks
--------------

This charm creates a base set of checks in /etc/nagios/nrpe.d, including
check\_load, check\_users, check\_disk\_root. All of the options for these are
configurable but sensible defaults have been set in config.yaml.
For example to increase the alert threshold for number of processes:

juju set nrpe load="-w 10,10,10 -c 25,25,25"

Principle Requested Checks
--------------------------

Monitors passed to this charm by the principle charm via the local-monitors
or nrpe-external-master relation. The principle charm can write its own
check definition into */etc/nagios/nrpe.d* and then inform this charm via the
monitors setting. It can also request a direct external check of a service
without using nrpe. See "Monitors yaml" below for examples.

User Requested Checks
---------------------

This works in the same way as the Principle requested except the monitors yaml
is set by the user via the monitors config option. For example to add a monitor
for the rsylog process:

    juju set nrpe monitors="
    monitors:
        local:
            procrunning:
                rsyslogd:
                    min: 1
                    max: 1
                    executable: rsyslogd
    "



External Nagios 
===============

If the nagios server is not deployed in the juju environment then the charm can
be configured, via the export\_nagios\_definitions, to write out nagios config
fragments to /var/lib/nagios/export. Rsync is then configured to allow a host
(specified by nagios\_master) to collect the fragments. An rsync stanza is created
allowing the Nagios server to pick up configs from /var/lib/nagios/export (as
a target called "external-nagios"), which will also be configured to allow
connections from the hostname or IP address as specified for the
"nagios\_master" variable.

It is up to you to configure the Nagios master to pull the configs needed, which
will then cause it to connect back to the instances in question to run the nrpe
checks you have defined.

Monitors yaml
=============

The list of monitors past down the monitors relation is an amalgamation of the
lists provided via the principle, the user and the default checks.

The monitors yaml is of the following form:

     
    # Version of the spec, mostly ignored but 0.3 is the current one
    version: '0.3'
    # Dict with just 'local' and 'remote' as parts
    monitors:
        # local monitors need an agent to be handled. See nrpe charm for
        # some example implementations
        local:
            # procrunning checks for a running process named X (no path)
            procrunning:
                # Multiple procrunning can be defined, this is the "name" of it
                nagios3:
                    min: 1
                    max: 1
                    executable: nagios3
        # Remote monitors can be polled directly by a remote system
        remote:
            # do a request on the HTTP protocol
            http:
                nagios:
                    port: 80
                    path: /nagios3/
                    # expected status response (otherwise just look for 200)
                    status: 'HTTP/1.1 401'
                    # Use as the Host: header (the server address will still be used to connect() to)
                    host: www.fewbar.com
            mysql:
                # Named basic check
                basic:
                    username: monitors
                    password: abcdefg123456
            nrpe:
                apache2:
                    command: check_apache2



Before a monitor is added it is checked to see if it is in the 'local' section.
If it is this charm needs to convert it into an nrpe checks. Only a small
number of check types are currently supported (see below) .These checks can
then be called by the nagios charm via the nrpe service. So for each check
listed in the local section:



1.  The defintion is read and a check defintion it written /etc/nagios/nrpe.d
2.  The check is defined as a remote nrpe check in the yaml passed to nagios

In the example above a check\_proc\_nagios3\_user.cfg file would be written
out which contains:

    # Check process nagios3 is running (user)
    command[check_proc_nagios3_user]=/usr/lib/nagios/plugins/check_procs -w 1 -c 1 -C nagios3

and the monitors yaml passed to nagios would include:

    monitors:
        nrpe:
	    check_proc_nagios3_user:
	        command: check_proc_nagios3_user

The principle charm, or the user via the monitors config option, can request an
external check by adding it to the remote section of the monitors yaml. In the
example above direct checks of a webserver and of mysql are being requested.
This charm passes those on to nagios unaltered.

Local check types
-----------------

Supported nrpe checks are:
    procrunning:
      min: Minimum number of 'executable' processes
      max: Maximum number of 'executable' processes
      executable: Name of executable to look for in process list
    processcount
      min: Minimum total number processes
      max: Maximum total number processes
      executable: Name of executable to look for in process list
    disk
      path: Directory to monitor space usage of

Remote check types
------------------

Supported remote types:
    http, mysql, nrpe, tcp, rpc, pgsql
    (See Nagios charm for up-to-date list and options)
