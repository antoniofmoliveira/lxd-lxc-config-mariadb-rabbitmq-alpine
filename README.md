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

    # get the ip address of running mariadbserver
    lxc list | grep mariadbserver | awk '{print $6}' | sed -r 's/\([^)]*\)//'
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

    get the ip address of running rabbitmqserver
    lxc list | grep rabbitmqserver | awk '{print $6}' | sed -r 's/\([^)]*\)//'
    http://the_ip_address:15672/

## Ubuntu Server as DevContainer

### set and start container for general use

    lxc launch ubuntu:24.04 first
    lxc config set first limits.memory=500MiB
    lxc config set first limits.cpu=4
    lxc config device override first root size=1GiB
    lxc restart first
    lxc info first
    lxc exec first -- bash
        adduser user
        usermod -aG sudo "user"
        apt instal openssh-server
        nano /etc/ssh/sshd_config
            PubkeyAuthentication yes
            AuthorizedKeysFile .ssh/authorized_keys
        systemctl restart sshd
        sudo user
            cd ~
            mkdir .ssh
            touch .ssh/authorized_keys
            nano .ssh/authorized_keys
                paste the pub key
            chmod 600 .ssh/authorized_keys
            chmod 700 .ssh
            exit
        exit

### access server

    ssh user@server

### dev container with LXD

    # get the ip address of running first
    lxc list | grep first | awk '{print $6}' | sed -r 's/\([^)]*\)//'
    - in vscode use extension remote-ssh (ms-vscode-remote.remote-ssh) to connect to server
    - then vscode server will be installed in the server
    - create a folder in terminal 
    - then open the folder in vscode
    - go to extensions and install the extension you want

## MongoDB Community Edition

### set and start mongodbserver

    lxc launch ubuntu:24.04 mongodbserver
    lxc config set mongodbserver limits.memory=500MiB
    lxc config set mongodbserver limits.cpu=4
    lxc config device override mongodbserver root size=1GiB
    lxc restart mongodbserver
    lxc info mongodbserver

### install mongodb

    lxc exec mongodbserver -- bash
        wget https://repo.mongodb.org/apt/ubuntu/dists/noble/mongodb-org/8.0/multiverse/binary-amd64/mongodb-org-server_8.0.3_amd64.deb -O mongodb.deb
        wget https://downloads.mongodb.com/compass/mongodb-mongosh_2.3.4_amd64.deb -O mongosh.deb
        dpkg -i mongodb.deb
        dpkg -i mongosh.deb
        systemctl enable mongod
        systemctl start mongod
        nano /etc/mongod.conf
            # network interfaces
            net:
                port: 27017
                bindIp: mongodbserver
                # bindIp: 127.0.0.1
            security:
                authorization: enabled
        systemctl restart mongod
        mongosh
            use db
            db.createUser(
                {
                    user: "use",
                    pwd: "password", // or passwordPrompt()
                    customData: { employeeId: 12345 },
                    roles: [
                        { role: "readWrite", db: "db" },
                    ]
                }
            )
        exit
    exit

### connect to mongodb

    # get the ip address of runnning mongodbserver
    lxc list | grep mongodbserver | awk '{print $6}' | sed -r 's/\([^)]*\)//'

    or

    export MONGODB_IP=$(lxc list | grep mongodbserver | awk '{print $6}' | sed -r 's/\([^)]*\)//')

    mongosh -u user -p password $MONGODB_IP/db --authenticationDatabase=db

    mongodb://user:password@the_ip_address:27017/?authSource=db
