Prerequisites
To proceed with this guide, ensure you have a set of three Ubuntu 20.04 nodes in the same Vultr location, configured with private networking. This guide uses the following private IP addresses for the nodes.

node-1 : Primary node PRIMATY_NODE_IP
node-2 : Replica node REPLICA_NODE1_IP
node-3 : Replica node REPLICA_NODE2_IP


# 1. Verify Redis Installation on All Servers
SSH to all three servers and follow the steps below to make sure the Redis server is installed and active.

Launch the redis-cli package.

     redis-cli
Run the ping command.

     127.0.0.1:6379> PING
If the Redis server is running, you should get the following output.

     127.0.0.1:6379> PONG
Press CTRL + C to log out from the Redis server.

After you have verified the Redis instances are up and running, you can now configure replication settings on them.


# 2. Configure Redis Server Replication Settings
The Redis server relies on many configuration settings to work in replication mode. You'll have to configure these settings in all the three nodes.

By default, you can locate the Redis configuration settings in the following file.

    /etc/redis/redis.conf
Use nano to open the file on all three servers for editing purposes.

    sudo nano /etc/redis/redis.conf*
Make the following configuration changes in all three servers. Remember to replace EXAMPLE_REPL_PASSWORD with a strong password.

node-1 : Primary node - PRIMATY_NODE_IP:

    bind 127.0.0.1 ::1 PRIMATY_NODE_IP
    protected-mode no
    requirepass EXAMPLE_REPL_PASSWORD

node-2 : Replica node - REPLICA_NODE1_IP:

    bind 127.0.0.1 ::1 REPLICA_NODE1_IP
    protected-mode no
    replicaof PRIMATY_NODE_IP 6379
    masterauth EXAMPLE_REPL_PASSWORD
    requirepass EXAMPLE_REPL_PASSWORD

node-3 : Replica node - REPLICA_NODE2_IP:

    bind 127.0.0.1 ::1 REPLICA_NODE2_IP
    protected-mode no

    replicaof PRIMATY_NODE_IP 6379
    masterauth EXAMPLE_REPL_PASSWORD

    requirepass EXAMPLE_REPL_PASSWORD
Save and close the file when you're through with editing.

Then, execute the following command to restart the Redis server in all three servers.

    sudo systemctl restart redis-server
You've correctly configured the Redis instances in all the nodes. In the next step, you'll test the new configuration settings.


# 3. Test Redis Replication
The following diagram illustrates the Redis replication architecture that you've set up.

    +------------------+      +---------------+  +---------------+
    |     Primary      | ---> |   Replica 1   |  |   Replica 2   |
    | (receive writes) |      |  (exact copy) |  |  (exact copy) |
    +------------------+      +---------------+  +---------------+
Every key you write to the primary Redis server will be automatically copied to the replica instances. To confirm this, execute the following steps.

Log in to the Redis server on the primary node-1 (PRIMATY_NODE_IP).

    redis-cli
Next, authenticate to the Redis server.

    127.0.0.1:6379> AUTH EXAMPLE_REPL_PASSWORD
Execute the following command to view the replication status.

    127.0.0.1:6379> info replication
You should get the following output.

 ### Replication
 role:master
 connected_slaves:2
 slave0:ip=REPLICA_NODE1_IP,port=6379,state=online,offset=1120,lag=0
 slave1:ip=REPLICA_NODE2_IP,port=6379,state=online,offset=1120,lag=0
 master_replid:fa1b59b5bf10563aaefd6b77d9ce5b455746506e
 master_replid2:0000000000000000000000000000000000000000
 master_repl_offset:1120
 second_repl_offset:-1
 repl_backlog_active:1
 repl_backlog_size:1048576
 repl_backlog_first_byte_offset:1
 repl_backlog_histlen:1120
Next, set up a test key with a value of, The key value was set on the primary node..

    127.0.0.1:6379> set test "The key value was set on the primary node."
Ensure you get the confirmation message below.

 OK
Opening two new terminal windows. Then, SSH to the replica nodes: node-2 (REPLICA_NODE1_IP) and node-3 (REPLICA_NODE2_IP).

    redis-cli
Authenticate to the Redis servers on the replica nodes.

    127.0.0.1:6379> AUTH EXAMPLE_REPL_PASSWORD
Attempt retrieving the value of the test key that you've created on the primary node.

    127.0.0.1:6379> GET test
You should get the following output showing the replication process is working as expected.

 "The key value was set on the primary node."


# 4. Promoting a Replica to Primary
In case the primary node goes down, you can promote any standby replica node to become the new primary using the following process.

Log in to the Redis server on the replica node. For instance, node-1 (REPLICA_NODE1_IP).

    redis-cli
Authenticate to the Redis server on node-1.

    127.0.0.1:6379> AUTH EXAMPLE_REPL_PASSWORD*
Instruct the replica to stop replicating data from the old primary node.

    127.0.0.1:6379> replicaof no one
Log out from the Redis server on the replica node.

    127.0.0.1:6379> QUIT;
Open the Redis configuration file on the replica node and remove the following lines to make Redis run in primary mode.

    replicaof PRIMATY_NODE_IP 6379
    masterauth EXAMPLE_REPL_PASSWORD
Save and close the file. Then, restart the Redis server on the new primary.

    sudo systemctl restart redis-server
Next, log in to the Redis server on node-2 (REPLICA_NODE2_IP). Authenticate and run the command below to instruct the remaining replica node to start replicating from the newly elected primary.

    127.0.0.1:6379> replicaof REPLICA_NODE1_IP 6379
Output.

 OK
On node-2 (REPLICA_NODE2_IP), execute the following command to make sure you've adjusted the new primary settings accordingly.

    127.0.0.1:6379> info replication
You should get the following output showing that the new primary is up and running.

 ### Replication
 role:slave
 master_host:REPLICA_NODE1_IP
 master_port:6379
 master_link_status:up
 master_last_io_seconds_ago:6
 master_sync_in_progress:0
 slave_repl_offset:42
 slave_priority:100
 slave_read_only:1
 connected_slaves:0
 master_replid:0612cd32ed6f06ca81b4ab21a6cff76b7e561b3e
 master_replid2:0000000000000000000000000000000000000000
 master_repl_offset:42
 second_repl_offset:-1
 repl_backlog_active:1
 repl_backlog_size:1048576
 repl_backlog_first_byte_offset:1
 repl_backlog_histlen:42

From this point forward, you can set the new primary IP address in any app that you'd previously connected to the old primary node.

