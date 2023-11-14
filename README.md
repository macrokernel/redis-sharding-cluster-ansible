# Ansible role for deployment of a sharding Redis Cluster in Docker Swarm

This Ansible role deploys a Redis Cluster with N shards and N replicas using the [official Redis Docker image](https://hub.docker.com/_/redis) and requires 2 * N Docker Swarm nodes. See more on Redis Cluster at https://redis.io/docs/management/scaling/.  

Redis clients are supposed to run in Docker Swarm and connect to the Redis Cluster via `redis` Docker Swarm overlay network using internal Docker DNS names, ex. `redis-master-1.redis`. A client connected to the overlay network may discover Redis Cluster hosts via DNS, for example, by using the following shell commands:
```shell
$ getent hosts tasks.redis-master
10.16.0.183     tasks.redis-master
10.16.0.182     tasks.redis-master
10.16.0.181     tasks.redis-master
$ getent hosts tasks.redis-slave
10.16.0.184     tasks.redis-slave
10.16.0.186     tasks.redis-slave
10.16.0.185     tasks.redis-slave
```

The hosts for deployment of Redis Cluster master and slave instances can be specified in `defaults/main.yml`. If fewer masters and slaves are required than there are hosts available, set `redis_replication_factor` in `defaults/main.yml`. You may set `docker_mtu` to a value lower than the default MTU 1500 if your LAN network is a VXLAN or other sort of a virtual network with a non-standard MTU size.

Example:
```yaml
redis_master_hosts:
  - redis-master-1.example.com
  - redis-master-2.example.com
  - redis-master-3.example.com
  - redis-master-4.example.com
redis_slave_hosts:
  - redis-slave-1.example.com
  - redis-slave-2.example.com
  - redis-slave-3.example.com
  - redis-slave-4.example.com
redis_replication_factor: 3
docker_mtu: 1390
```

Redis configuration parameters can be specified in `templates/redis.conf.j2`. Here is the default contents of the file:
```
port 6379
timeout 0
tcp-keepalive 300
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

Aside of N master and N replica Redis Cluster docker containers this Ansible role deploys a helper container named `redis-cluster-initiator-1`, which, when started, waits for all Redis master and slave containers to come online, then joins them into a Redis Cluster. The helper container keeps running and logs Redis Cluster node status once in a minute:
``` shell
$ docker service logs -f redis-cluster-initiator
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | >>> Cluster nodes
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | 851a1113734c61f1f04e41f92322389243d02915 10.16.0.181:6379@16379 master - 0 1699459444155 2 connected 5461-10922
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | 1717a7aa2114c4da28bf15273c19c6d1becbbe91 10.16.0.186:6379@16379 slave 028c64b6ef6dcf340e69ee93addb35d118732a53 0 1699459444054 1 connected
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | a0853bb4c291667b845b71110d1f6d4fb244ccdc 10.16.0.182:6379@16379 master - 0 1699459444558 3 connected 10923-16383
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | 198d1dfc3bba76494b9448ca02e63b8577d8eea8 10.16.0.185:6379@16379 slave a0853bb4c291667b845b71110d1f6d4fb244ccdc 0 1699459443551 3 connected
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | 028c64b6ef6dcf340e69ee93addb35d118732a53 10.16.0.183:6379@16379 myself,master - 0 1699459444000 1 connected 0-5460
redis-cluster-initiator.1.4gtwsif3gnrb@docker-swarm-3  | 171ec91eec4cbf1e4e0cd91e68ac6137e1b5f3e3 10.16.0.184:6379@16379 slave 028c64b6ef6dcf340e69ee93addb35d118732a53 0 1699459444000 1 connected
```
