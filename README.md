# Redis Cluster Playground Guide

## The Playground (Quickstart)

Redis Cluster Playground helps you get a live Redis Cluster running on localhost with the least amount of typing possible.

Running a Redis Cluster requires, at a minimum:

- Multiple Redis instances running in cluster mode
- Telling the Redis instances to join together as a unified cluster

Redis Cluster Playground does both of those for you in two simple commands.

### Shorter Intro

There's a short meta-readme at my [Cluster Intro Nov 2013](https://matt.sh/short-redis-cluster-intro-nov-2013) page.

### Prerequisites

- Have <code>tmux</code> installed.

- Make sure <code>redis-server</code> and <code>redis-trib.rb</code> are in your path.
  - Redis Cluster is still in development.  You should clone <code>https://github.com/antirez/redis</code>, check out the <code>unstable</code> branch, compile it, then add it to your <code>PATH</code> for the duration of this tutorial.

- Clone the Redis Cluster Playground repository:
<pre>
git clone https://github.com/mattsta/redis-cluster-playground
</pre>

### Set Up Cluster Nodes

#### Create 24 Redis instances running in cluster mode on localhost
<pre>
./play.sh nodes 24
</pre>

After a brief pause, you'll see a Redis server running in your shell.  There are 23 other Redis servers running as well.  You have a <code>tmux</code> session open holding all 24 Redis instances in cluster mode. (Note: they aren't a cluster yet.)

If you look across the bottom of your terminal, you'll see the names of the other running Redis servers.  You can quickly pick another server to look at with <code>Ctrl-B w</code> or cycle through them with <code>Ctrl-B p</code> for previous or <code>Ctrl-B n</code> for next.

To exit all Redis instances, just hold down <code>Ctrl-C</code>.  When one server exits, the next running server takes its place in your terminal.  Holding down <code>Ctrl-C</code> will quickly kill them all and exit <code>tmux</code>.

To detach <code>tmux</code> from your terminal, hit <code>Ctrl-B + d</code>.  To reattach to your session, run <code>tmux attach</code>.

### Connect the Cluster Nodes

#### Turn 24 Redis instances into a Redis Cluster

##### Create a 24-instance cluster of only master nodes
In another terminal, run:
<pre>
./play.sh create 24
</pre>

Read the output (note the number of "master" nodes) and type 'yes' to initialize your localhost cluster.

This is a special cluster because a.) it's running on localhost, b.) every instance in this cluster is a master and c.) this cluster is not saving any data (everything is in memory only).  Since this cluster is full of *only* master nodes with no redundancy, if any node goes down, your entire cluster becomes unusable.


##### Create a 24-instance cluster of 8 master nodes and 16 slave nodes
To avoid an all-or-nothing cluster, you can create a cluster with two replicas for each master instance:
<pre>
./play.sh create-with-replicas 24 2
</pre>

Now you have three copies of your data in the cluster (one copy at a master node, and two copies at each slave for that master).

NOTE: You can only run <code>create</code> or <code>create-with-replicas</code> *once* per cluster.  If you want to re-create the cluster: kill all Redis instances, re-run <code>./play.sh nodes N</code>, then re-run the <code>create</code> or <code>create-with-replicas</code> command.


## Play with the cluster

You now have a Redis Cluster running on your machine!  Pick an instance and connect to it with <code>redis-cli</code>:
<pre>
redis-cli -p 7011
</pre>

Looks normal so far, doesn't it?

Now try to write something:
<pre>
127.0.0.1:7011> set mykey myvalue
(error) MOVED 14687 127.0.0.1:7003
</pre>

Whoops.  We're running a cluster, and cluster instances can *only* accept writes to the slots they currently own (slot ownership is assigned at cluster creation time—go back and look at your <code>create</code> output to see the assigned ranges per instance).

Redis Cluster doesn't proxy your requests, so it's kindly telling you to go check with instance 7003 to run your command against <code>mykey</code>.  If you try many different keys, you'll notice they each map to various master instances in your cluster.

Hop over to instance 7003 and try your command again:
<pre>
% redis-cli -p 7003
127.0.0.1:7003> set mykey myvalue
OK
127.0.0.1:7003> get mykey
"myvalue"
</pre>

Success!

Now you can play around with the abilities and limitations of Redis Cluster.  Try setting a few keys, reading a few keys, and killing a master node to watch a slave promotion.  Play around with the various options of <code>redis-trib.rb</code> to re-shard and add nodes to your cluster.


## Check Cluster Nodes

Redis Cluster Playground lets you iterate over your cluster instances and check their status easily.

To check one instance, give its port on localhost:
<pre>
./play.sh check 7011
</pre>

or, check all your instance by giving the number of instances you passed to <code>nodes</code>:
<pre>
./play.sh checks 24
</pre>

Checking all your instances is only useful if you haven't run <code>create</code> against yet.  Once your cluster is connected, running <code>check</code> on one will retrieve information from all other instances as well.



## Behind The Scenes (What Redis Cluster Playground is Doing For You)

### Individual Cluster Nodes
To run a cluster, you need multiple <code>redis-server</code> processes running in cluster mode.

Each <code>redis-server</code> process must be started in cluster mode.  Stand alone or Sentinel managed primary/secondary nodes are not allowed to participate in a cluster.

Enable cluster mode by setting <code>cluster-enabled yes</code> and assigning a <code>cluster-config-file [node-specific-name].conf</code> in your <code>redis.conf</code>.  (Reminder: all redis configuration options are valid command line arguments as well.)  You can get a quick memory-only localhost cluster node up and running with:
<pre>
redis-server --bind 127.0.0.1 --port 1110 --save '' --cluster-enabled yes --cluster-config-file node-110.conf
</pre>


### The Cluster Itself

You can create a cluster of <code>redis-server</code> processes using the <code>redis-trib.rb</code> command.

<code>redis-trib.rb</code> is a high level coordination/management/setup program that uses Redis low level cluster primitives to get your cluster running quickly (assigning slots, assigning backup instances, retarding, etc).

To get your cluster up and running, run:
<pre>
redis-trib.rb create host0:port0 host1:port1 host2:port2 ...
</pre>

#### Important Note About Node Types

In a Redis Cluster, an instance is either a <code>master</code> or <code>slave</code> instance.  By default, <code>redis-trib.rb</code> assigns all instances to a <code>master</code> role.  This means if *any* instance goes down or becomes unreachable, your entire cluster gets disabled.  You can enable auto-allocation of slave nodes by adding <code>--replicas N</code> after <code>create</code> but before your list of hosts.  The replica count means *each master* gets N hot spare instances.  The backup/slave instances can be used for reads, but not writes (until they get promoted to master themselves).


#### Node Math
If you want 64 master nodes each with N=3 redundancy, then you'll need 256 nodes total (64 masters + 3 slaves * 64 masters = 256 total nodes).  Also note: in this case, a redundancy of three gives you *four* copies of your data in the cluster, since you are starting three replicas per master, and the master already has the data.

Other node math scenarios:

  - 12 total nodes with replica level of 1 = 6 master nodes, 6 slave nodes.
  - 12 total nodes replica level of 2 = 4 master nodes, 8 slave nodes.
  - 12 total nodes with replica level of 3 = 3 master nodes, 9 slave nodes.
  - 12 total nodes with replica level of 4 = impossible to configure.  A minimum of 15 total nodes would be needed.


General formulas for instance counts:

  - Number of Master Nodes = (Number of Total Nodes) / (Replica Count + 1)
  - Total Nodes Required = Number of Master Nodes * (Replica Count + 1)
  - [alternatively] Total Nodes Required = Number of Master Nodes + Number of Master Nodes * Replica Count
  - [trivial] Number of Slave Nodes = Number of Total Nodes - Number of Master Nodes

If you want one slave ("read only / hot spare backup") for each master node, your total instance count doubles.  If you have 12 total nodes and ask <code>redis-trib.rb</code> to assign a replica level of one slave node per master node, you end up with 6 master nodes and 6 slave nodes.

Each slave node is responsible for a *specific* master node.  The slaves do not handle multiple datasets from multiple masters.

You can (and should) run multiple Redis instances on an individual machine.  You can run multiple master nodes and multiple slave nodes on an individual machine.  Redis Cluster provisioning tools will be clever enough to not assign slave to a master on the same machine.

You can think of Redis instances as vnodes in other systems.  By default, <code>redis-trib.rb</code> uses the IP address of instances to help determine how instances should be configured.  Two instances on the same IP address won't be backups for each other (unless you don't have enough variety in your nodes to spread your requested load around, like running 60 nodes all on 127.0.0.1).


## Final Notes

### Purpose

Redis Cluster Playground is meant as an exploratory tool.  Do not use Redis Cluster Playground to deploy production services.  Do not use Redis Cluster Playground as a persistent backing store.  Do not taunt Redis Cluster Playground.

### Contributions

We're just getting started.  Issues and pull requests are welcome.
