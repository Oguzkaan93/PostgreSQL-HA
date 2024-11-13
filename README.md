# PostgreSQL-HA
8 Nodes configuration of PostgreSQL - Patroni - ETCD - Keepalive - Haproxy - Pgbackrest - PgBouncer Cluster

This projects assumes you have installed Oracle Linux 8.10 on Oracle VM VirtualBox

Nodes IP:  
Node1 192.168.68.100  
Node2 192.168.68.101  
Node3 192.168.68.102  
Node4 192.168.68.103  
Node5 192.168.68.104  
Node6 192.168.68.105  
Node7 192.168.68.106  
Node8 192.168.68.107  

On Nodes 1-2-3
Patroni - PostgreSQL

On Nodes 4-5-6
ETCD - PgBouncer

On Node 7
Pgbackrest - HaProxy - Keepalived

On Node 8
HaProxy - Keepalived

# Server Configurations

* Make server ip static for each server  
  nmcli connection modify enp0s3 ipv4.addresses "$Node_Ip/24" ipv4.gateway "192.168.68.1" ipv4.dns "1.1.1.1" ipv4.method manual  
  nmcli connection down enp0s3  
  nmcli connection up enp0s3  
* Enable Epel Repo  
  yum install epel-release -y  
* Stop firewall and disable selinux  
  systemctl stop firewalld.service  
  systemctl disable firewalld.service  
  setenforce 0  
  vim /etc/selinux/  
  Change "SELINUX=enforcing" to  "SELINUX=disabled"  
* Configure passwordless SSH with each server  
  ssh-keygen -t rsa  
  ssh-copy-id $target_host_ip
# Install PostgreSQL on nodes 1-2-3-7  
yum install postgresql16-server postgresql16-libs.x86_64 postgresql16-devel.x86_64 postgresql16-contrib.x86_64 -y  
### If you get error like below:  
Error:
 Problem: cannot install the best candidate for the job
 nothing provides perl(IPC::Run) needed by postgresql16-devel-16.4-1PGDG.rhel8.x86_64 from pgdg16
 (try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)

You need to activate ol8_codeready_builder repo  
vim /etc/yum.repos.d/oracle-linux-ol8.repo  
Change "enable=0" to "enable=1" in repo [ol8_codeready_builder]

### Configure passwordless SSH and add postgres user to sudoers file on Nodes 1-2-3-7  
ssh-keygen -t rsa  
ssh-copy-id $target_host_ip  
sudo visudo  
At the end of the file add  
"postgres        ALL=(ALL)       NOPASSWD: ALL"  
# Installing ETCD on nodes 4-5-6  
Following guide in https://github.com/etcd-io/etcd/releases/tag/v3.5.16  
  
ETCD_VER=v3.5.16  
  
Choose either URL  
GOOGLE_URL=https://storage.googleapis.com/etcd  
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download  
DOWNLOAD_URL=${GOOGLE_URL}  
  
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz  
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test  
  
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz  
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1  
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz  
  
/tmp/etcd-download-test/etcd --version  
/tmp/etcd-download-test/etcdctl version  
/tmp/etcd-download-test/etcdutl version  
  
* Move etcd, etcdctl and etcdutl files to path /usr/bin  
  mv /tmp/etcd-download-test/etcd* /usr/bin/.  

* Create etcd service file  
  vim /etc/systemd/system/etcd.service  
***Attached file {ETCD Service file}***  
sudo systemctl daemon-reload  
  
sudo mkdir -p /var/lib/etcd/  
sudo mkdir /etc/etcd  
sudo groupadd --system etcd  
sudo useradd -s /sbin/nologin --system -g etcd etcd  
sudo chown -R etcd:etcd /var/lib/etcd/  
sudo chmod -R a+rw /var/lib/etcd  
sudo chown etcd:etcd /usr/bin/etcd*  

* Edit ETCD config file  
    ***Attached file {ETCD Config File}***  
sudo chown etcd:etcd /etc/etcd/etcd.conf  
# Install patroni on nodes 1-2-3-7 
following guide at https://github.com/patroni/patroni  
  
Install pip  
python3 -m ensurepip --default-pip  
  
vim /var/lib/pgsql/.bash_profile then add	
export PATH=$PATH:/var/lib/pgsql/.local/bin  
  
pip install patroni[psycopg3,etcd3]  
  
python3 -m pip install --upgrade pip  
  
mkdir -p /etc/patroni  
vim /etc/patroni/postgresql.yml  
 ***Attached file {Postgresql.yml}***  
chown -R postgres:postgres /etc/patroni  
  
vim /etc/systemd/system/patroni.service  
  ***Attached file {patroni.service}***  
systemctl daemon-reload  
systemctl start patroni.service  
  
# Install pgbackrest on nodes 1-2-3-7  
yum install pgbackrest -y  
chown postgres:postgres /etc/pgbackrest.conf  
  
## On Nodes 1-2-3  
vim /etc/pgbackrest.conf  
  ***Attached file {PostgreSQL Node config file}***  
## On Node 7  
vim /etc/pgbackrest.conf  
  ***Attached file {Repo config file}***  
pgbackrest --stanza=patroni_stanza stanza-create  
pgbackrest --stanza=patroni_stanza check --log-level-console=detail  
pgbackrest --stanza=patroni_stanza backup --log-level-console=detail  
  
# Install haproxy and keepalived on node 7-8  
  
yum install haproxy  
cp haproxy.cfg haproxy.cfg.bck  
vim haproxy.cfg  
  ***Attached file {Haproxy config file}***  
systemctl start haproxy.service  
  
yum install keepalived -y  
cp  /etc/keepalived/keepalived.conf.bckp  
vim /etc/keepalived/keepalived.conf  
  ***Attached file {Keepalive config file}***  
systemctl start keepalived.service  
  
# Install pgbouncer on node 4-5-6  
  
yum install pgbouncer -y  
chown -R pgbouncer:pgbouncer pgbouncer/  
cp /etc/pgbouncer/pgbouncer.ini /etc/pgbouncer/pgbouncer.ini.bckp  
vim /etc/pgbouncer/pgbouncer.ini  
  ***Attached file {Pgbouncer config file}***  
psql -Atq -h 192.168.68.230 -p 5433 -U postgres -d postgres -c "SELECT concat('\"', usename, '\" \"', passwd, '\"') FROM pg_shadow" >> /etc/pgbouncer/userlist.txt  
systemctl start pgbouncer.service  
