# PostgreSQL-HA
8 Nodes configuration of PostgreSQL - Patroni - ETCD - Keepalive - Haproxy - Pgbackrest - PgBouncer Cluster

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
Patroni - PostgreSQL - PgBouncer 

On Nodes 4-5-6
ETCD

On Node 7
Pgbackrest - HaProxy - Keepalived

On Node 8
HaProxy - Keepalived
