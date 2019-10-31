# mariadb_galera
mariadb galera master master cluster installation based on ubuntu 18.04 and Vagrant

This reposiroty contains a Vagrantfile that creates 5 nodes:
- 2 haproxy/keepalived servers
- 3 mariadb servers configured as galera master-master cluster

To install the solution you can install execute a command:
  vagrant up
or install all instances one-by-one:
  a) install mysql nodes
  vagrant up mysql-node1 --provision
  vagrant up mysql-node2 --provision
  vagrant up mysql-node3 --provision
  
  b) install keepalived and haproxy nodes
  vagrant up srv1 --provision
  vagrant up srv2 --provision
  


