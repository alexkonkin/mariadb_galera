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

$install_mysql_cluster = <<-SCRIPT
   nod_nam=$1
   nod_ip=$2
   is_first_node=$3

   sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
   sudo add-apt-repository 'deb [arch=amd64] http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.4/ubuntu bionic main'
   sudo apt update
   sudo apt install mariadb-server rsync mc -y
   mysql -uroot -e 'set password = password("123456");'

   echo -e "172.16.94.11       mysql-node1\n172.16.94.12       mysql-node2\n172.16.94.13       mysql-node3" >> /etc/hosts

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

  sudo systemctl start mysql

  mysql -u root --password=123456 -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "mysql1" do |mysql1|
    mysql1.vm.box = "bento/ubuntu-18.04"
    mysql1.vm.hostname = "mysql-node1"
    mysql1.vm.network :private_network, ip: "192.168.1.11"
    mysql1.vm.network :private_network, ip: "172.16.94.11"
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node1","172.16.94.11","true"]
    end
  end

  config.vm.define "mysql2" do |mysql2|
    mysql2.vm.box = "bento/ubuntu-18.04"
    mysql2.vm.hostname = "mysql-node2"
    mysql2.vm.network :private_network, ip: "192.168.1.12"
    mysql2.vm.network :private_network, ip: "172.16.94.12"
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node2","172.16.94.12","false"]
    end
  end

  config.vm.define "mysql3" do |mysql3|
    mysql3.vm.box = "bento/ubuntu-18.04"
    mysql3.vm.hostname = "mysql-node3"
    mysql3.vm.network :private_network, ip: "192.168.1.13"
    mysql3.vm.network :private_network, ip: "172.16.94.13"
    #mysql3.vm.provision "shell", path: 'provisioners/install_k8s.sh', privileged: false
    config.vm.provision "shell" do |script|
       script.inline = $install_mysql_cluster
       script.args = ["mysql-node3","172.16.94.13","false"]
    end
   end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
