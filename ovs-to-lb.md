STOP RPCDAEMON IN AN RPC ENVIRONMENT!

NOTE: All work should be done from the controller/network node unless specified otherwise.
P.S. Be sure to use the correct Neutron DB user/pass where applicable

## Backup your databases just in case something goes terribly wrong ##
mkdir /root/ovs-lb-upgradprep/
mysqldump --defaults-extra-file=/root/.my.cnf --routines --single-transaction --all-databases | gzip > /root/ovs-lb-upgradprep/ovs2lb-db-backup-$(date +%s).sql.gz

## Install required preliminary packages ##
dsh -Mg all "apt-get update"
dsh -Mg all "apt-get install -y bridge-utils vlan"

## Take note of OVS state just because ##
dsh -Mg all "echo \$(hostname); ovs-vsctl show; echo; brctl show; echo" | tee -a ovs-state-show.txt

## Create a migration database. Prep the database for changes ##
neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini stamp havana
mysqldump neutron > /root/ovs-lb-upgradprep/neutron-ovs.sql
mysql -e "CREATE DATABASE neutron_ml2_migration;"
mysql -e "GRANT ALL PRIVILEGES ON neutron_ml2_migration.* TO 'neutron'@'localhost';"
mysql -e "GRANT ALL PRIVILEGES ON neutron_ml2_migration.* TO 'neutron'@'%';"
mysql neutron_ml2_migration < /root/ovs-lb-upgradprep/neutron-ovs.sql

## Pull down database migration script and execute on migration database ##
wget https://raw.githubusercontent.com/busterswt/openstack-networking/master/migrate_to_ml2_mod.py
SQL_CON=$(echo -n `grep mysql /etc/neutron/neutron.conf | awk '{print $3}'` && echo '_ml2_migration')
python -m migrate_to_ml2_mod --tunnel-type gre --release havana openvswitch $SQL_CON

## Update migration database with new VLANs for each network and then dump the resulting DB ##
#Network:  1197fa26-b10f-44d0-afd3-a0a90bb0ead5
mysql -e "update neutron_ml2_migration.ml2_network_segments set physical_network='physnet1' where network_id='1197fa26-b10f-44d0-afd3-a0a90bb0ead5';"
mysql -e "update neutron_ml2_migration.ml2_network_segments set segmentation_id='203' where network_id='1197fa26-b10f-44d0-afd3-a0a90bb0ead5';"
mysql -e "update neutron_ml2_migration.ml2_network_segments set network_type='vlan' where network_id='1197fa26-b10f-44d0-afd3-a0a90bb0ead5';"
mysql -e "insert into neutron_ml2_migration.ml2_vlan_allocations (physical_network,vlan_id,allocated) values ('physnet1','203','1');"

## Repeat the above steps for each network/vlan you are converting from GRE -> VLAN ##

mysql -e "update neutron_ml2_migration.ml2_port_bindings set vif_type='bridge',driver='linuxbridge' where vif_type='ovs';"
mysql -e "truncate neutron_ml2_migration.ml2_gre_endpoints;"

mysqldump neutron_ml2_migration > /root/ovs-lb-upgradprep/neutron-ml2-MIGRATED.sql

## Backup config files ##
dsh -Mg controller "cp -a /etc/nova/nova.conf{,.bak}"
dsh -Mg controller "cp -a /etc/neutron/dhcp_agent.ini{,.bak}"
dsh -Mg controller "cp -a /etc/neutron/neutron.conf{,.bak}"
dsh -Mg controller "cp -a /etc/neutron/l3_agent.ini{,.bak}"
dsh -Mg controller "cp -a /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini{,.bak}"
dsh -Mcg compute "cp -a /etc/nova/nova.conf{,.bak}"
dsh -Mcg compute "cp -a /etc/neutron/neutron.conf{,.bak}"
dsh -Mcg compute "cp -a /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini{,.bak}"

## Upgrade the python-six and stevedore packages ##
dsh -Mg all "apt-get install -y python-pip"
dsh -Mg all "pip install stevedore --upgrade" #should be v1.0.0
dsh -Mg all "pip install six==1.4.1"
(make sure that stevedore is v1.0.0 and six is 1.4.1)

## Create the new neutron_ml2 database manually. It's not a necessary step, but helps keep the migration DB separated from our new production DB ##
SQL_PASS=$(echo $SQL_CON | cut -d ':' -f3 | cut -d '@' -f1)
mysql -e "CREATE DATABASE neutron_ml2 character set utf8;"
mysql -e "GRANT ALL PRIVILEGES ON neutron_ml2.* TO 'neutron'@'localhost' IDENTIFIED BY '$SQL_PASS';"
mysql -e "GRANT ALL PRIVILEGES ON neutron_ml2.* TO 'neutron'@'%' IDENTIFIED BY '$SQL_PASS';"

## Import the neutron_ml2_migration DB ##
mysql neutron_ml2 < /root/ovs-lb-upgradprep/neutron-ml2-MIGRATED.sql

## Remove the physical port from the provider OVS bridge and remove dhcp/router ports from integration bridge. Provider bridge and interface may vary ##
dsh -Mg all "ovs-vsctl del-port br-eth0 eth0"
dsh -Mg controller "for PORT in \$(ovs-vsctl show | grep \"qr\|qg\" | grep Port | awk '{print \$2}' | sed 's/\"//g'); do ovs-vsctl del-port br-int \$PORT; done"
dsh -Mg controller "for PORT in \$(ovs-vsctl show | grep -E 'Port.*tap' | awk '{print \$2}' | sed 's/\"//g'); do ovs-vsctl del-port br-int \$PORT; done"

## Disable OVS-related services ##
dsh -Mcg all "update-rc.d neutron-plugin-openvswitch-agent disable; update-rc.d openvswitch-switch disable"
dsh -Mcg all "echo manual > /etc/init/neutron-plugin-openvswitch-agent.override; echo manual > /etc/init/openvswitch-switch.override"

## Stop services ##
dsh -Mcg all "service neutron-plugin-openvswitch-agent stop"
dsh -Mcg all "service openvswitch-switch stop; rmmod openvswitch"h
dsh -Mg controller "service neutron-server stop; service neutron-dhcp-agent stop; service neutron-lbaas-agent stop; service neutron-l3-agent stop"

## Remove instance tap interfaces from the OVS-related linux bridges ##
dsh -Mg all "for TAPNAME in \$(brctl show | grep tap); do brctl delif qbr\${TAPNAME:3:14} \${TAPNAME:0:14}; done"

## Replace OVS driver with LinuxBridge driver for DHCP and L3 agents ##

dsh -Mg controller "sed -i 's/^interface_driver.*$/interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver/' /etc/neutron/dhcp_agent.ini"
dsh -Mg controller "sed -i 's/^interface_driver.*$/interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver/' /etc/neutron/l3_agent.ini"

## Configure Nova compute to use LinuxBridge driver so taps plugin into brq* bridges ##

dsh -Mg all "sed -i 's/^libvirt_vif_driver.*$/libvirt_vif_driver=nova.virt.libvirt.vif.NeutronLinuxBridgeVIFDriver/' /etc/nova/nova.conf"
dsh -Mg all "sed -i 's/^linuxnet_interface_driver.*$/linuxnet_interface_driver=nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver/' /etc/nova/nova.conf"

## Misc Neutron configuration including configuring Neutron to use ML2 plugin and L3 service plugin. ##
## If using FWaaS or LBaaS, be sure to ass that service plugin to the list ##

dsh -Mg controller "sed -i 's/^connection.*$/connection = mysql:\/\/neutron:neutron@controller\/neutron_ml2/' /etc/neutron/neutron.conf"
dsh -Mg all "sed -i '/^ovs_use_veth.*$/d' /etc/neutron/neutron.conf"
dsh -Mg all "sed -i 's/^core_plugin.*$/core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin\nservice_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin/g' /etc/neutron/neutron.conf"

## Install the LinuxBridge agent ##
dsh -Mg all "apt-get -y install neutron-plugin-linuxbridge-agent"

## Add ML2 and LinuxBridge agent configuration files ##
## Note: Use the same provider label to ensure no changes are needed for existing Neutron networks ##

dsh -Mg all "mkdir -m 750 /etc/neutron/plugins/ml2; chown root:neutron /etc/neutron/plugins/ml2"
dsh -Mg all "mkdir -m 750 /etc/neutron/plugins/linuxbridge; chown root:neutron /etc/neutron/plugins/linuxbridge"

SQL_CON=$(echo -n `grep -m1 mysql /etc/neutron/neutron.conf | awk '{print $3}'`;)

cat >> /etc/neutron/plugins/ml2/ml2_conf.ini << EOF
[ml2]
mechanism_drivers = linuxbridge
type_drivers = vlan
tenant_network_types = vlan

[agent]

[ml2_type_vlan]
network_vlan_ranges = physnet1

[database]
sql_connection = $SQL_CON
sqlalchemy_pool_size = 10
reconnect_interval = 2

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
EOF

cat >> /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini << EOF
[vlans]
tenant_network_type = vlan
network_vlan_ranges = physnet1

[linux_bridge]
physical_interface_mappings = physnet1:eth0

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
EOF

## Copy configuration files to all hosts ##

for NODE in $(cat /etc/dsh/group/all | grep -v $(hostname -s)); do scp /etc/neutron/plugins/ml2/ml2_conf.ini $NODE:/etc/neutron/plugins/ml2/ml2_conf.ini; done
dsh -Mg all "chmod 640 /etc/neutron/plugins/ml2/ml2_conf.ini; chown root:neutron /etc/neutron/plugins/ml2/ml2_conf.ini"
for NODE in $(cat /etc/dsh/group/all | grep -v $(hostname -s)); do scp /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini $NODE:/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini; done
dsh -Mg all "chmod 640 /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini; chown root:neutron /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini"

## Change the plugin file referenced in /etc/default/neutron-server (default is OVS) ##

dsh -Mg controller "sed -i 's/^NEUTRON_PLUGIN_CONFIG.*$/NEUTRON_PLUGIN_CONFIG=\"\/etc\/neutron\/plugins\/ml2\/ml2_conf.ini\"/' /etc/default/neutron-server"

## Restart services ##
dsh -Mg controller "for i in /etc/init.d/neutron*; do \$i restart; done"
dsh -Mg compute "service neutron-plugin-linuxbridge-agent restart"
dsh -Mg compute "service nova-compute restart"

## If you have double taps for DHCP namespaces, fix DHCP namespaces by bringing down the *old* tap interfaces ##
## After reboot this will be fixed ##
dsh -Mg controller "for NETNS in \$(ip netns | grep qdhcp); do TAP=\$(ip netns exec \$NETNS ip a | grep tap -m 1 | awk '{print \$2}' | sed s/://); ip netns exec \$NETNS ip link set \$TAP down; done"

## Test connectivity within the environment ##
for i in $(nova list | awk '/ACTIVE/ {print $12}' | cut -d '=' -f2); do ip netns exec qdhcp-<NET-ID> ping -c 3 $i ; done

## Test connectivity out of the environment ##
ip netns exec qdhcp-<NET-ID> ssh <user>@x.x.x.x
ping -c 3 google.com
exit
nova delete <id>

** NOTE **
Do not remove OVS bridges, packages or reboot any of the nodes until the environment has been confirmed operational
This is really the only point at which we can easily roll back. Any changes to the Neutron DB (ie new instances/ports) will be lost in case of a rollback.

## Once the migration has been deemed a success, clean up ##
dsh -Mg all "apt-get -y remove neutron-plugin-openvswitch neutron-plugin-openvswitch-agent openvswitch-common openvswitch-datapath-dkms openvswitch-switch"
for AGENT in $(neutron agent-list | awk '/Open vSwitch/ {print $2}'); do neutron agent-delete $AGENT; done

