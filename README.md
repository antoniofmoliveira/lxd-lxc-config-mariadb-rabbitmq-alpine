# LXD with MariaDB and RabbitMQ in linux containers using Alpine

## LXD install

    sudo apt update
    sudo apt install snapd
    sudo snap install lxd
    sudo snap refresh lxd
    getent group lxd | grep "$USER"
    sudo usermod -aG lxd "$USER"
    newgrp lxd
    lxd init --minimal

## via browser (LXD UI)

    http://localhost:8443

## MariaDB

### set and start container

    lxc image list images: alpine 3.20 architecture=$(uname -m)
    lxc launch images:alpine/3.20 mariadbserver
    lxc config set mariadbserver limits.memory=500MiB
    lxc config set mariadbserver limits.cpu=4
    lxc config device override mariadbserver root size=1GiB
    lxc restart mariadbserver
    lxc info mariadbserver
    lxc storage list

### install mariadb

    lxc exec mariadbserver -- sh
        apk update
        apk add mariadb mariadb-common mariadb-client
        rc-service mariadb setup
        # rc-service mariadb start
        rc-update add mariadb default
        mariadb-secure-installation

### add the following at the end of /etc/my.cnf.d/mariadb-server.cnf

    [mysqld]
    skip-networking=0
    skip-bind-address

### configure mariadb

    lxc exec mariadbserver -- sh
        mysql -u root -p
            create database somedb;
            create user 'user'@'%' identified by 'password';
            grant all privileges on somedatabase to 'user'@'%';
    lxc snapshot mariadbserver mariadbserver-init

### connect to mariadb

    lxc list # get the ip address of mariadbserver
    mysql -h the_ip_address -u user -p

## RabbitMQ

### set and start container for rabbitmq

    lxc launch images:alpine/3.20 rabbitmqserver
    lxc config set rabbitmqserver limits.memory=500MiB
    lxc config set rabbitmqserver limits.cpu=4
    lxc config device override rabbitmqserver root size=1GiB
    lxc restart rabbitmqserver
    lxc info rabbitmqserver
    
### install rabbitmq

    lxc exec rabbitmqserver -- sh
        apk update
        apk add rabbitmq-server
        #rc-service rabbitmq-server start
        rc-update add rabbitmq-server default
        mkdir /etc/rabbitmq
        touch /etc/rabbitmq/enabled_plugins
        rabbitmq-plugins enable rabbitmq_management
        rabbitmqctl add_user username password
        rabbitmqctl set_user_tags username administrator, management
        rabbitmqctl set_permissions username ".*" ".*" ".*"
    lxc snapshot rabbitmqserver rabbitmqserver-init

### via browser (RabbitMQ Management UI)

    lxc list # get the ip address of rabbitmqserver
    http://the_ip_address:15672/
