# 区块链平台 docker搭建步骤

## 环境
mkdocker
ubuntu:14.04 基础上制作


## Docker In Docker
```
docker run   --name node1  --net="host" --privileged -t -i  -d ubuntu:14.04
```

1.更新软件源

```
vi /etc/apt/sources.list
http://mirrors.163.com/.help/ubuntu.html
https://mirrors.tuna.tsinghua.edu.cn/

deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse
```

2.安装  docker
```
docker
https://docs.docker.com/engine/installation/linux/ubuntulinux/

apt-get install apt-transport-https ca-certificates -y
apt-key adv \
               --keyserver hkp://ha.pool.sks-keyservers.net:80 \
               --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

echo "deb https://apt.dockerproject.org/repo ubuntu-trusty main" | sudo tee /etc/apt/sources.list.d/docker.list
apt-get update
apt-cache policy docker-engine
    option
        apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual


apt-get install docker-engine -y
service docker start

docker commit -m='ubuntu with docker' --author='allen' node1 ubuntu:docker-1404
```

## 安装vim 
DockerFile
```
#Dockerfile
FROM ubuntu:docker-1404
MAINTAINER Allen.Liu

#setup
RUN  apt-get install -y vim
```

*** docker build -t ubuntu:docker-vim-1404  . ***

## 安装mysql 
DockerFile
```
#Dockerfile
FROM ubuntu:docker-vim-1404
MAINTAINER Allen.Liu

#setup
RUN echo "mysql-server-5.5 mysql-server/root_password password root" | debconf-set-selections
RUN echo "mysql-server-5.5 mysql-server/root_password_again password root" | debconf-set-selections
RUN apt-get install -y mysql-server

#enable access

#CMD ["mysqld_safe"]
```
*** docker build -t ubuntu:docker-vim-mysql-1404  .  ***


## 自启动服务制作
supervisord.conf

```
[supervisord]
nodaemon=true

[program:mysqld]
command=/usr/bin/mysqld_safe

[program:docker]
command=/bin/bash -c "service docker start"
```

DockerFile
```
#Dockerfile
FROM ubuntu:docker-vim-mysql-1404
MAINTAINER Allen.Liu
RUN apt-get install -y supervisor
RUN mkdir -p /var/log/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
CMD ["/usr/bin/supervisord"]
```

*** docker build -t daap:run  . ***


## 测试
```
docker save -o daap-run.tar daap:run
tar -zcvf daap-run.tar.gz  daap-run.tar
tar -xvzf daap-run.tar.gz
docker load -i daap-run.tar
docker run   --name node1  --privileged -t -i  -d  daap:run
```


需要运行在docker  1.12 版本以上, 宿主机安装mysql可能导致冲突(docker in docker + mysql 环境启动,本地安装mysql启动可能会冲突)



default 私钥
```
        {"privateKey":"626f0352f0d1308ccef6986670951dccc2c86477489023326c761591282bbb01","publicKey":"03e7be8c8c8f490f01df8b07b99b75eac94e2d6d512415b99e411ce224c306c788","address":"911529b57f887f34947353c8b35bc58d7c8e097c","bind":""}
        
curl -X POST  http://ip:8545  --data  '{"jsonrpc":"2.0","method":"eth_getBalance","params":["911529b57f887f34947353c8b35bc58d7c8e097c","latest"],"id":1}'
```



## mysql 5.7
```
docker run   --name node1  --net="host" --privileged -t -i  -d ubuntu:docker-vim-1404
wget http://dev.mysql.com/get/mysql-apt-config_0.8.0-1_all.deb
dpkg -i mysql-apt-config_0.8.0-1_all.deb
apt-get update
docker commit -m='ubuntu with docker mysql57' --author='allen' node1 ubuntu:docker-mysql57-1404
```

```
apt-get install mysql-server
docker commit -m='ubuntu with docker mysql57' --author='allen' node1 ubuntu:docker-mysql57source-1404
```

DockerFile
```
FROM ubuntu:docker-mysql57-1404
MAINTAINER allen
RUN apt-get install -y supervisor
RUN mkdir -p /var/log/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
CMD ["/usr/bin/supervisord"]
```


## 公有链 
生产环境节点接入  net host方式 确保宿主机8080端口未被占用

DockFile
```
FROM daap:run2
MAINTAINER Allen.liu

ENV PDX_DAAP_HOME /home
ENV JAVA_HOME /home/jdk1.8.0_65

RUN mkdir /var/daap && \
    mkdir /home/webdocker && \
    mkdir /home/data && \
    mkdir /home/data/d_apps && \
    mkdir /home/.ethash && \
    ln -s /home/.ethash /root/.ethash

ADD pdx_web.tar /home/webdocker
ADD daap-container.tar /home/
ADD daap-bc.tar /home/
ADD conf.tar /home/
ADD jdk1.8.0_65.tar /home/
COPY .bashrc /root/

COPY chaininfo /home/
COPY startup.sh /home/daap-container/bin/
COPY startBc.sh /home/daap-container/bin/

EXPOSE 8080
```

startup.sh
```
di=`docker images|grep pdx_web_docker|wc -l`
 if [  $di -eq 0 ]
        then
        echo "Web Docker loading"
        docker load -i /home/webdocker/pdx_web_docker.tar
        echo "Web Docker done"
 else
    echo "Web Docker loaded already"
fi
```

startBc.sh
```
#!/bin/sh

DAAP_BC_HOME=$PDX_DAAP_HOME/daap-bc
cd $PDX_DAAP_HOME/daap-container/bin
nohup $DAAP_BC_HOME/ethchain1  --datadir $DAAP_BC_HOME/data --genesis $DAAP_BC_HOME/genesis.json --networkid 888 --rpc    --rpcaddr 0.0.0.0  -rpccorsdomain "*"  > ../logs/bc.log 2>&1 &
echo "Block chain protocol stack started."
```
## 私有链

私有节点 

```
默认私钥匙
priKey:72ff615aa9c5559869ad3624cdaf19361572be4abdf9d8e790a3e1079f04960c
pubkey:03537b140e63261cad598c5a746725c0c3bd5c9ad62ba2f142fc63a3c7c317df18
address:0581fb775c2a43bc5949fc4bd133f503badabcd9
nodeid
enode://f06e934c6022f118f0690301e5c1833b8f345b413cecc1b446b6c22ca9d22979aec8a0421ab332fd4b7e25d6e1437c0975abbdcb2b118104ff828e1b696aa021@[::]:30303
```

json sed 替换
```
    sed -i 's/\("privateKey":"\).*/\1bbbbb"\,/g'
    sed 's#\("privateKey":"\).*#"privateKey":"aaaabbbb"\,#' testfile
    sed -i "s/^privateKey=.*/privateKey=$newPrivateKey/g" $PDX_DAAP_HOME/conf/pdxdaap.props
    sed -i "s/\(\"privateKey\"\:\"\).*/\1$newPrivateKey\"\,/g" $PDX_DAAP_HOME/chaininfo
    sed -i "s/\(\"X-DaaP-NodeId\"\:\"\).*/\1$newAddress\"\,/g" $PDX_DAAP_HOME/chaininfo
```


DockerFile
```
FROM daap:run2
MAINTAINER Allen.liu

ENV PDX_DAAP_HOME /home
ENV JAVA_HOME /home/jdk1.8.0_65

RUN mkdir /var/daap && \
    mkdir /home/webdocker && \
    mkdir /home/data && \
    mkdir /home/data/d_apps && \
    mkdir /home/.ethash && \
    ln -s /home/.ethash /root/.ethash

ADD pdx_web.tar /home/webdocker
ADD daap-container.tar /home/
ADD daap-bc.tar /home/
ADD conf.tar /home/
ADD jdk1.8.0_65.tar /home/
COPY .bashrc /root/

COPY chaininfo /home/
COPY setupSeed.sh /home/daap-container/bin/
COPY setupNode.sh /home/daap-container/bin/
COPY startup.sh /home/daap-container/bin/
COPY startBc.sh /home/daap-container/bin/
ADD gastools.tar /home/daap-container/bin/
EXPOSE 8080

```

setupSeed.sh
```
#!/bin/sh

#
#set miner
#
    BCHOME=$PDX_DAAP_HOME/daap-bc
    echo "You want to set your  private block chain,please follow instructions as below:"
    miner=`grep minerthread $PDX_DAAP_HOME/daap-container/bin/startBc.sh |wc -l`
    if [ $miner -eq 0 ]
    then

#
#update startup scripts
#
    sed -i "s/^nohup \$DAAP_BC_HOME\/ethchain1/& \-\-mine \-\-minerthreads \"1\" /g"     $PDX_DAAP_HOME/daap-container/bin/startBc.sh
    echo "set miner done."
    fi

#
#read and update networkid
#
#   echo -n "Please enter your network id(The default is 888):" 
#   read networkid  
#   networkid=${networkid:=888}
#   echo "set networkid done."
#
#   sed -i "s/\-\-networkid.*\-\-rpc[ ]/\-\-networkid $networkid \-\-rpc  /g" $PDX_DAAP_HOME/daap-container/bin/startBc.sh  

#
#generate DAG
#
    
    if [ -f $HOME/.ethash/full-R23-0000000000000000 ] 
    then
        echo "DAG  has been  generated already, skip this step."
        else
         echo "Generate DAG,it might takes a while."
         $PDX_DAAP_HOME/daap-bc/ethchain1 makedag 100 $HOME/.ethash 
         echo "DAG  generated done."

        echo "Seed Node setup has been done succesfully!, you can run startup.sh to get daap container running"
    fi

```

setupNode.sh
```
#!/bin/sh



#
#generate key pair
#

    newKeyPair=`java -jar ./gastools/ethgastools.jar -gen `
    newPrivateKey=`echo $newKeyPair|awk '{print $2}'`
    newPublicKey=`echo $newKeyPair|awk '{print $4}'`
    newAddress=`echo $newKeyPair|awk '{print $6}'`
    echo "newPrivateKey = $newPrivateKey" > $PDX_DAAP_HOME/conf/.nodekeypair
    echo "newPublicKey =  $newPublicKey" >> $PDX_DAAP_HOME/conf/.nodekeypair
    echo "newAddress = $newAddress" >> $PDX_DAAP_HOME/conf/.nodekeypair

#
#update pdxdaap.props
#
    #sed -i "s/^privateKey=.*/privateKey=$newPrivateKey/g" $PDX_DAAP_HOME/conf/pdxdaap.props
    sed -i "s/\(\"privateKey\"\:\"\).*/\1$newPrivateKey\"\,/g" $PDX_DAAP_HOME/chaininfo
        sed -i "s/\(\"X-DaaP-NodeId\"\:\"\).*/\1$newAddress\"\,/g" $PDX_DAAP_HOME/chaininfo


    BCHOME=$PDX_DAAP_HOME/daap-bc
    echo "Please enter ip address or host name of your private blockchain's seed?"
    read seedHost

#
#write info into static-nodes.json
#
    echo "[" > $BCHOME/data/static-nodes.json
    echo "\"enode://f06e934c6022f118f0690301e5c1833b8f345b413cecc1b446b6c22ca9d22979aec8a0421ab332fd4b7e25d6e1437c0975abbdcb2b118104ff828e1b696aa021@$seedHost:30303\"" >> $BCHOME/data/static-nodes.json
    echo "]" >> $BCHOME/data/static-nodes.json

#
#inform user input network id
#
#   echo -n "Please enter your network id(The default is 888):"
#   read networkid
#   networkid=${networkid:=888}

#
#update networkid into 
#

#   sed -i "s/\-\-networkid.*\-\-rpc[ ]/\-\-networkid $networkid \-\-rpc  /g" $PDX_DAAP_HOME/daap-container/bin/startBc.sh
#   echo "Node setup almost done!, please be patient."
    

#
#offer gas
#
    java -jar ./gastools/ethgastools.jar -ip $seedHost -privatekey 00ee6fe31778be27ece49de289f051064555012b47b70d55906c785e3390e628d2 -address $newAddress

    sleep 30s
    balance=`java -jar ./gastools/ethgastools.jar -ip $seedHost  -getbalance -address $newAddress`  
    echo "balance = $balance"

#
#remove director and file under /home/daap-bc/data 
#
    rm -fr /home/daap-bc/data/chaindata
    rm -fr /home/daap-bc/data/dapp
    rm -fr /home/daap-bc/data/geth.ipc
    rm -fr /home/daap-bc/data/nodes
    rm -fr /home/daap-bc/data/nodekey
    echo "Configuration Done! You can run daap container right now!"

```

