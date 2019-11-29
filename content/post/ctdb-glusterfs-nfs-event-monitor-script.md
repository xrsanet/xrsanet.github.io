---
title: "CTDB Glusterfs NFS Event Monitor Script"
date: 2015-04-25T17:10:35Z
draft: false
categories: ["Sysadmin"]
---
There is no CTDB monitor script for the GlusterFS NFS implementation as you cannot use the normal NFS event script that comes with CTDB, this is because GlusterFS manages NFS.

Without a proper monitoring script CTDB will not initiate a failover when NFS services fail, below I have created a script to solve this problem.

# Code

{{< highlight bash >}}
#!/bin/sh
# Event script to monitor GlusterFS NFS in a cluster environment
# Source: https://xrsa.net
# Author: Ben Draper
# Email: ben@xrsa.net
# Install Location: /etc/ctdb/events.d/60.glusternfs
#
 
[ -n "$CTDB_BASE" ] || \
    export CTDB_BASE=$(cd -P $(dirname "$0") ; dirname "$PWD")
 
. $CTDB_BASE/functions
 
service_ports="111 2049 38465 38466"
 
# Placeholder
service_name="glusterfs_nfs"
 
# Verify GlusterFS Services are running
verify_supporting_services ()
{
    l_ret=0
    /usr/bin/systemctl -q is-active glusterd || {
        echo "Service - glusterd is not running"
        l_ret=1
    }
 
    /usr/bin/systemctl -q is-active glusterfsd || {
        echo "Service - glusterfsd is not running"
        l_ret=1
    }
 
    /usr/bin/systemctl -q is-active rpcbind || {
        echo "Service - rpcbind is not running"
        l_ret=1
    }
 
    if [ $l_ret -eq 1 ]; then
        exit $l_ret
    fi
}
 
# This will verify required ports are listening
verify_ports ()
{
    l_ret=0
    for check_port in $service_ports; do
        ctdb_check_tcp_ports $check_port || {
            l_ret=1
        }
    done
 
    if [ $l_ret -eq 1 ]; then
        exit $l_ret
    fi
}
 
loadconfig
case "$1" in
    monitor)
        verify_supporting_services
        verify_ports    
        update_tickles 2049
    ;;
 
    *)
	ctdb_standard_event_handler "$@"
    ;;
esac
 
exit 0
{{< /highlight >}}

# CentOS 7 – SELinux

I am assuming you already have SELinux working and configured for your GlusterFS / CTDB environment. If so, you will need to add an additional policy for my event script to work properly.

{{< highlight bash >}}
# cat > CTDB_GlusterNFS.te << EOF
 
module CTDB_NFS 1.0;
 
require {
        type portmap_port_t;
        type nfs_port_t;
        class tcp_socket name_bind;
	type rpcd_unit_file_t;
	type systemd_unit_file_t;
	type ctdbd_t;
	class service status;
}
 
#============= ctdbd_t ==============
allow ctdbd_t rpcd_unit_file_t:service status;
allow ctdbd_t systemd_unit_file_t:service status;
allow ctdbd_t nfs_port_t:tcp_socket name_bind;
allow ctdbd_t portmap_port_t:tcp_socket name_bind;
EOF
 
# checkmodule -M -m -o CTDB_GlusterNFS.mod CTDB_GlusterNFS.te
# semodule_package -o CTDB_GlusterNFS.pp -m CTDB_GlusterNFS.mod
# semodule -i CTDB_GlusterNFS.pp
{{< /highlight >}}