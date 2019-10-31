VAGRANTFILE_API_VERSION = "2"

$install_mysql_cluster1 = <<-SCRIPT
  nod_nam=$1
  nod_ip=$2
  is_first_node=$3

  echo $nod_nam
  echo $nod_ip
  echo $is_first_node

  sed -i 's/nm/'$nod_nam'/g' /etc/mysql/conf.d/galera.cnf
  sed -i 's/node_ip/'$nod_ip'/g' /etc/mysql/conf.d/galera.cnf
SCRIPT

$install_keepalived_haproxy = <<-SCRIPT
  state=$1
  priority=$2
  this_ip=$3
  peer_ip=$4

  echo $nod_nam
  echo $nod_ip
  echo $is_first_node
  sudo apt install keepalived haproxy mc -y
  echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n172.16.94.11       mysql-node1\n172.16.94.12       mysql-node2\n172.16.94.13       mysql-node3" >> /etc/hosts

cat > /etc/haproxy/haproxy.cfg << "END"
global
    log 127.0.0.1 local0 notice
    user haproxy
    group haproxy


systemctl stop haproxy
systemctl stop keepalived

defaults
    log     global
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  50000
    timeout server  50000
    stats enable
    stats uri /haproxy/stats
    stats auth admin:admin


frontend http
    bind *:80
    mode http
    option httplog
    default_backend webservers

backend webservers
    mode http
    stats enable
    stats uri /haproxy/stats
    stats auth admin:admin
    stats hide-version
    balance roundrobin
    option httpclose
    option forwardfor
    cookie SRVNAME insert
    server srv1 192.168.1.11:80 check cookie srv1
    server srv2 192.168.1.12:80 check cookie srv2

frontend sql
    bind 192.168.1.100:3306
    mode tcp
    option tcplog
    default_backend dbservers

backend dbservers
    mode tcp
    balance leastconn
    option tcpka
    server mysql-node1 172.16.94.11:3306 check weight 1
    server mysql-node2 172.16.94.12:3306 check weight 1
    server mysql-node3 172.16.94.13:3306 check weight 1
END

cat > /etc/keepalived/keepalived.conf << "END"
vrrp_script chk_haproxy {
    script "killall -0 haproxy"     # cheaper than pidof
    interval 2                      # check every 2 seconds
}

vrrp_instance VI_1 {
    state STATE
    interface eth1
    virtual_router_id 51
    priority PRIORITY
    vrrp_unicast_bind THIS_IP   # Internal IP of this machine
    vrrp_unicast_peer PEER_IP   # Internal IP of peer
    virtual_ipaddress {
        192.168.1.100 dev eth1 label eth1:vip1
    }
    track_script {
        chk_haproxy weight 2
    }
}
END

#allow binding to virtual IP of keepalived and apply this adjustment
echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
sysctl -p

#configue master/slave keepalived instances
sed -i 's/STATE/'$state'/g' /etc/keepalived/keepalived.conf
sed -i 's/PRIORITY/'$priority'/g' /etc/keepalived/keepalived.conf
sed -i 's/THIS_IP/'$this_ip'/g' /etc/keepalived/keepalived.conf
sed -i 's/PEER_IP/'$peer_ip'/g' /etc/keepalived/keepalived.conf

systemctl start keepalived
systemctl start haproxy

SCRIPT

$install_mysql_cluster = <<-SCRIPT
   nod_nam=$1
   nod_ip=$2
   is_first_node=$3

   sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
   sudo add-apt-repository 'deb [arch=amd64] http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.4/ubuntu bionic main'
   sudo apt update
   sudo apt install mariadb-server rsync mc -y
   mysql -uroot -e 'set password = password("123456");'

   echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n172.16.94.11       mysql-node1\n172.16.94.12       mysql-node2\n172.16.94.13       mysql-node3" >> /etc/hosts

cat > /etc/mysql/conf.d/galera.cnf << "END"
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="test_cluster"
wsrep_cluster_address="gcomm://172.16.94.11,172.16.94.12,172.16.94.13"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="node_ip"
wsrep_node_name="nm"
END
#sed -i 's/node_name/'$(hostname)'/g' /etc/mysql/conf.d/galera.cnf
#sed -i 's/node_ip/'$(hostname -I|awk '''{print $3}''')'/g' /etc/mysql/conf.d/galera.cnf
  echo $nod_nam
  echo $nod_ip
  echo $is_first_node

  sed -i 's/nm/'$nod_nam'/g' /etc/mysql/conf.d/galera.cnf
  sed -i 's/node_ip/'$nod_ip'/g' /etc/mysql/conf.d/galera.cnf

  sudo systemctl stop mysql

  if [ $is_first_node == true ];then
    sudo galera_new_cluster
  fi

  #allow MariaDb to listen on network interface instead of localhost
  sed -i 's/bind-address\t\t= 127.0.0.1/#bind-address\t\t= 127.0.0.1\nskip-bind-address\nskip-networking=0\n/g' /etc/mysql/my.cnf

  sudo systemctl start mysql

  #enable connections from the network with haproxy instances
  mysql -u root --password=123456 -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.1.%' IDENTIFIED BY '123456' WITH GRANT OPTION;"
  mysql -u root --password=123456 -e "FLUSH PRIVILEGES;"

  mysql -u root --password=123456 -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
SCRIPT


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "srv1" do |srv1|
    srv1.vm.box = "bento/ubuntu-18.04"
    srv1.vm.hostname = "srv1"
    srv1.vm.network :private_network, ip: "192.168.1.11"
    config.vm.provision "shell" do |script|
       script.inline = $install_keepalived_haproxy
       script.args = ["MASTER","101","192.168.1.11","192.168.1.12"]
    end
  end

  config.vm.define "srv2" do |srv2|
    srv2.vm.box = "bento/ubuntu-18.04"
    srv2.vm.hostname = "srv2"
    srv2.vm.network :private_network, ip: "192.168.1.12"
    config.vm.provision "shell" do |script|
       script.inline = $install_keepalived_haproxy
       script.args = ["BACKUP","100","192.168.1.12","192.168.1.11"]
    end
  end

  config.vm.define "mysql1" do |mysql1|
    mysql1.vm.box = "bento/ubuntu-18.04"
    mysql1.vm.hostname = "mysql-node1"
    mysql1.vm.network :private_network, ip: "172.16.94.11"
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node1","172.16.94.11","true"]
    end
  end

  config.vm.define "mysql2" do |mysql2|
    mysql2.vm.box = "bento/ubuntu-18.04"
    mysql2.vm.hostname = "mysql-node2"
    mysql2.vm.network :private_network, ip: "172.16.94.12"
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node2","172.16.94.12","false"]
    end
  end

  config.vm.define "mysql3" do |mysql3|
    mysql3.vm.box = "bento/ubuntu-18.04"
    mysql3.vm.hostname = "mysql-node3"
    mysql3.vm.network :private_network, ip: "172.16.94.13"
    #mysql3.vm.provision "shell", path: 'provisioners/install_k8s.sh', privileged: false
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node3","172.16.94.13","false"]
    end
   end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "320"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
