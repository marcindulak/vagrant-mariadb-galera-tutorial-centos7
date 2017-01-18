# -*- mode: ruby -*-
# vi: set ft=ruby :

MARIADBVER="10.1"
SST_PASSWORD="SST_PASSWORD"

hosts = {
  'node0' => {'hostname' => 'node0', 'ip' => '192.168.10.10', 'mac' => '080027001010'},
  'node1' => {'hostname' => 'node1', 'ip' => '192.168.10.11', 'mac' => '080027001011'},
  'node2' => {'hostname' => 'node2', 'ip' => '192.168.10.12', 'mac' => '080027001012'},
  'client0' => {'hostname' => 'client0', 'ip' => '192.168.10.100', 'mac' => '080027000100'}
}

CONTROLLER = ENV.fetch('CONTROLLER', 'IDE Controller')

Vagrant.configure(2) do |config|
  hosts.keys.sort.each do |host|
    if host.start_with?("node")
      config.vm.define hosts[host]['hostname'] do |node|
        node.vm.box = 'centos/7'
        node.vm.box_url = 'centos/7'
        node.vm.synced_folder '.', '/home/vagrant/sync', disabled: true
        node.vm.network 'private_network', ip: hosts[host]['ip'], mac: hosts[host]['mac'], auto_config: false
        node.vm.provider 'virtualbox' do |v|
          v.memory = 256
          v.cpus = 1
          # disable VBox time synchronization and use ntp
          v.customize ['setextradata', :id, 'VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled', 1]
          # sdb block device is the database partition
          disk = hosts[host]['hostname'] + 'sdb.vdi'
          if !File.exist?(disk)
            v.customize ['createhd', '--filename', disk, '--size', 384, '--variant', 'Fixed']
            v.customize ['modifyhd', disk, '--type', 'writethrough']
          end
          v.customize ['storageattach', :id, '--storagectl', CONTROLLER, '--port', 0, '--device', 1, '--type', 'hdd', '--medium', disk]
          # sdc block device for Galera swap
          disk = hosts[host]['hostname'] + 'sdc.vdi'
          if !File.exist?(disk)
            v.customize ['createhd', '--filename', disk, '--size', 256, '--variant', 'Fixed']
            v.customize ['modifyhd', disk, '--type', 'writethrough']
          end
          v.customize ['storageattach', :id, '--storagectl', CONTROLLER, '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
        end
      end
    end
  end
  hosts.keys.sort.each do |host|
    if host.start_with?("client")
      config.vm.define hosts[host]['hostname'] do |client|
        client.vm.box = 'centos/7'
        client.vm.box_url = 'centos/7'
        client.vm.synced_folder '.', '/home/vagrant/sync', disabled: true
        client.vm.network 'private_network', ip: hosts[host]['ip'], mac: hosts[host]['mac'], auto_config: false
        client.vm.provider 'virtualbox' do |v|
          v.memory = 256
          v.cpus = 1
          # disable VBox time synchronization and use ntp
          v.customize ['setextradata', :id, 'VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled', 1]
        end
      end
    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop firewalld
  $systemctl_stop_firewalld = <<SCRIPT
systemctl stop firewalld.service
SCRIPT
  # common settings on all machines
  $etc_hosts = <<SCRIPT
echo "$*" >> /etc/hosts
SCRIPT
  $mariadb_el = <<SCRIPT
MARIADBVER=$1
if test `uname -m` == i686;
then
BASEARCH=x86
fi
if test `uname -m` == x86_64;
then
BASEARCH=amd64
fi
cat <<END > /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/$MARIADBVER/centos\\$releasever-$BASEARCH
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
END
SCRIPT
  $etc_my_cnf_d_server_cnf = <<SCRIPT
WSREP_NODE_ADDRESS=$1
WSREP_NODE_NAME=$2
PASSWORD=$3
NODES=$4
chown mysql.mysql /etc/my.cnf.d/server.cnf
chmod o-rwx /etc/my.cnf.d/server.cnf
sed -Ei "/Mandatory settings/iwsrep_node_name=$WSREP_NODE_NAME" /etc/my.cnf.d/server.cnf
sed -Ei "/Mandatory settings/iwsrep_node_address=$WSREP_NODE_ADDRESS" /etc/my.cnf.d/server.cnf
sed -Ei "/Mandatory settings/iwsrep_sst_method=rsync" /etc/my.cnf.d/server.cnf
sed -Ei "/Mandatory settings/iwsrep_sst_auth=sst:$PASSWORD" /etc/my.cnf.d/server.cnf
sed -Ei 's|#wsrep_on=.*|wsrep_on=ON|' /etc/my.cnf.d/server.cnf
sed -Ei 's|#wsrep_provider=.*|wsrep_provider=/usr/lib64/galera/libgalera_smm.so|' /etc/my.cnf.d/server.cnf
sed -Ei "s|#wsrep_cluster_address=.*|wsrep_cluster_address=gcomm://$NODES|" /etc/my.cnf.d/server.cnf
sed -Ei 's|#binlog_format=.*|binlog_format=row|' /etc/my.cnf.d/server.cnf
sed -Ei 's|#default_storage_engine=.*|default_storage_engine=InnoDB|' /etc/my.cnf.d/server.cnf
sed -Ei 's|#innodb_autoinc_lock_mode=.*|innodb_autoinc_lock_mode=2|' /etc/my.cnf.d/server.cnf
sed -Ei "s|#bind-address=.*|bind-address=$WSREP_NODE_ADDRESS|" /etc/my.cnf.d/server.cnf
SCRIPT
  $mysql_secure_installation = <<SCRIPT
#!/usr/bin/expect --
spawn mysql_secure_installation

expect "Enter current password for root (enter for none):"
send "\r"

expect "Set root password?"
send "y\r"

expect "New password:"
send "ROOT_PASSWORD\r"

expect "Re-enter new password:"
send "ROOT_PASSWORD\r"

expect "Remove anonymous users?"
send "y\r"

expect "Disallow root login remotely?"
send "y\r"

expect "Remove test database and access to it?"
send "y\r"

expect "Reload privilege tables now?"
send "y\r"

puts "Ended expect script."
SCRIPT
  $mysql_sst = <<SCRIPT
PASSWORD=$1
echo "GRANT USAGE ON *.* to sst@'%' IDENTIFIED BY '$PASSWORD';" | mysql
echo "GRANT ALL PRIVILEGES ON *.* to sst@'%';" | mysql
SCRIPT
  # configure the second vagrant eth interface
  $ifcfg = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
TYPE=$4
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=$IPADDR
NETMASK=$NETMASK
DEVICE=$DEVICE
PEERDNS=no
TYPE=$TYPE
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  hosts.keys.sort.each do |host|
    if host.start_with?("node")
      config.vm.define hosts[host]['hostname'] do |node|
        node.vm.provision :shell, :inline => 'hostname ' + hosts[host]['hostname'], run: 'always'
        hosts.keys.sort.each do |k|
          node.vm.provision 'shell' do |s|
            s.inline = $etc_hosts
            s.args   = [hosts[k]['ip'], hosts[k]['hostname']]
          end
        end
        node.vm.provision :shell, :inline => $setenforce_0, run: 'always'
        node.vm.provision 'shell' do |s|
          s.inline = $mariadb_el
          s.args   = [MARIADBVER]
        end
        node.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
        node.vm.provision 'shell' do |s|
          s.inline = $ifcfg
          s.args   = [hosts[host]['ip'], '255.255.255.0', 'eth1', 'Ethernet']
        end
        node.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
        # restarting network fixes RTNETLINK answers: File exists
        node.vm.provision :shell, :inline => 'systemctl restart network'
        node.vm.provision :shell, :inline => $linux_disable_ipv6, run: 'always'
        # prepare database partition
        node.vm.provision :shell, :inline => 'mkfs.ext4 -F /dev/sdb'
        node.vm.provision :shell, :inline => 'echo /dev/sdb /var/lib/mysql ext4 defaults >> /etc/fstab'
        node.vm.provision :shell, :inline => 'mkdir /var/lib/mysql&& mount -a'
        # add additional swap partition
        node.vm.provision :shell, :inline => 'mkswap /dev/sdc'
        node.vm.provision :shell, :inline => 'echo UUID=`blkid -o value /dev/sdc | head -n 1` swap swap defaults 0 0 >> /etc/fstab'
        node.vm.provision :shell, :inline => 'yum -y remove mariadb*'
        node.vm.provision :shell, :inline => 'yum -y install MariaDB galera'
        node.vm.provision :shell, :inline => 'yum -y install expect'
        node.vm.provision :shell, :inline => 'systemctl start mariadb'
        node.vm.provision 'shell' do |s|
          s.inline = $mysql_sst
          s.args   = [SST_PASSWORD]
        end
        node.vm.provision :shell, :inline => $mysql_secure_installation
        # deciding which node starts first is left to the human ...
        node.vm.provision :shell, :inline => 'systemctl stop mariadb'
        node.vm.provision :shell, :inline => 'systemctl disable mariadb'
        node.vm.provision :shell, :inline => 'chkconfig mysql off'
        node.vm.provision 'shell' do |s|
          s.inline = $etc_my_cnf_d_server_cnf
          s.args   = [hosts[host]['ip'], hosts[host]['hostname'], SST_PASSWORD, hosts.keys.sort.to_a.join(',').gsub('client0', '').gsub(/^,/, '').gsub(/,$/, '')]
        end
        # install and enable ntp
        node.vm.provision :shell, :inline => 'yum -y install ntp'
        node.vm.provision :shell, :inline => 'systemctl enable ntpd'
        node.vm.provision :shell, :inline => 'systemctl start ntpd'
      end
    end
  end
  hosts.keys.sort.each do |host|
    if host.start_with?("client")
      config.vm.define hosts[host]['hostname'] do |client|
        client.vm.provision :shell, :inline => 'hostname ' + hosts[host]['hostname'], run: 'always'
        hosts.keys.sort.each do |k|
          client.vm.provision 'shell' do |s|
            s.inline = $etc_hosts
            s.args   = [hosts[k]['ip'],  hosts[k]['hostname']]
          end
        end
        client.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
        client.vm.provision 'shell' do |s|
          s.inline = $ifcfg
          s.args   = [hosts[host]['ip'] , '255.255.255.0', 'eth1', 'Ethernet']
        end
        client.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
        # restarting network fixes RTNETLINK answers: File exists
        client.vm.provision :shell, :inline => 'systemctl restart network'
        client.vm.provision :shell, :inline => $linux_disable_ipv6, run: 'always'
        # install and enable ntp
        client.vm.provision :shell, :inline => 'yum -y install ntp'
        client.vm.provision :shell, :inline => 'systemctl enable ntpd'
        client.vm.provision :shell, :inline => 'systemctl start ntpd'
      end
    end
  end
end
