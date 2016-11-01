# Tempest: A real-time graph database (Beta)
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Features](#features)
- [Introduction](#introduction)
  - [Graphs are Everywhere](#graphs-are-everywhere)
  - [Applications of Graph Analysis](#applications-of-graph-analysis)
  - [Why a new Graph Database?](#why-a-new-graph-database)
  - [Our Approach](#our-approach)
  - [Created by](#created-by)
- [Using Tempest DB](#using-tempest-db)
  - [Recommended Machine Sizes](#recommended-machine-sizes)
- [Using Tempest as a library](#using-the-tempest-library)
  - [Requirements](#requirements)
  - [Converting a Graph to Tempest Format](#converting-a-graph-to-tempest-format)
  - [Language Support for Tempest](#language-support-for-tempest)
    - [Scala](#scala)
    - [Java](#java)
    - [Python](#python)

- [Project Roadmap](#project-roadmap)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Features
- Single node, scale-up architecture: Graphs are notoriously hard to partition if you also need to support real-time traversals. So Tempest runs on a single, large-ish memory node (see [here](#recommended-machine-sizes)).
- Scalable: Tempest supports graphs with billions of nodes and edges.  For example, it can store a 30 billion
  edge graph using 300GB of RAM.
- Fast:  Accessing a neighbor of a node is as fast as two RAM accesses. Tempest supports about 1000 edge additions per second  when storing them persistently, and hundreds of thousands  per second when storing in RAM only.
- Convenient: Tempest is installed with a single Docker command.  Tempest can also be used as a library (together with your favorite persistent store), and is available as a single sbt or Maven dependency.  
- Fast loading: Tempest loads its graph format instantly, even for 30-billion edge graphs, so you can
  quickly deploy new versions of your code.
- Flexible: Tempest DB supports clients in any language [Thrift](https://thrift.apache.org/)
  supports. We provide client wrappers for Python and Ruby.
  
## Introduction
### Graphs are everywhere
A graph database like Tempest enables you to store and access your graphs the way you conceptually
think of them: as nodes and their connections. Graphs don’t just mean social networks; graphs are 
everywhere. There are communication graphs such as messaging, Skype, and email. There are graphs of
users and products, with each purchase (or view or click) corresponding to an edge in the graph. 
There are interest graphs such as Twitter and Pinterest. There are collaboration graphs such as 
Wikipedia and, increasingly, Google Docs and Office 365. The Internet of Things (IoT) is really a 
graph of connected devices. These graphs are growing, and will continue to grow, both in scale and 
in diversity. Traditional data storage and analytics systems (e.g., SQL, Hadoop, Spark) are poorly 
suited to help you store, access and analyze your graphs. You need a graph database for it.

### Applications of Graph Analysis
Graphs are useful in a variety of search and recommendation problems, for example:
  - Given the search query "John" from a user Alice, find all users named John 
    who are friends-of-friends with Alice.
  - Find all products similar to the ones Jane has bought, where two products are similar if a large
    fraction of people who buy one also buy the other.

A number of graph databases out there perform very poorly on neighborhood and traversal queries. And that performance degrades rapidly as the graph scales. In fact, no graph database out there can be used for real-time queries at the scale of Twitter or Linkedin. That is, no graph database other than Tempest. 

Tempest was purpose-built to enable blazingly fast neighborhood queries. By blazingly fast, we mean in under 250 milliseconds for a graph with 100M nodes and 5B edges.  In fact, Tempest is about 15x faster at these queries than Neo4j. Tempest scales to graphs with up to 4B nodes and 50B edges. It supports 1000 writes (new nodes and edges) per second while supporting truly real-time queries.

### Why a new Graph Database?
At [Teapot](http://teapot.co) we work with large social networks, and we needed a graph library that
could efficiently allow real-time computation on graphs with 10s of billions of edges.  We wanted a
JVM-based library so we could write efficient high-level code in Scala.  Our web backend is in Ruby,
so we also needed a way for languages like Ruby or python to efficiently access our graph.  We
considered using a distributed library like GraphX or Giraph, but those libraries don't support
real-time queries, and the complexity and overhead of managing a cluster were undesirable.  On AWS,
the cost of a machine scales linearly with the amount of RAM used, so for the price of a 256GB
cluster, we could rent a single 256GB machine, and have the convenience and performance of running
Scala code directly on a single machine.  We initially used the [Cassovary](https://github.com/twitter/cassovary) library, 
but as our graph grew beyond a few billion edges, Cassovary was taking hours to load our graph,
making deployments difficult without significant downtime to our service.  This led us to rethink
how we store graphs.

### Our Approach
A key feature of Tempest is reading data directly from a memory mapped binary file, so there is no
parsing required and no object or garbage collector overhead.  Once the graph file is cached in RAM
by the OS, Tempest can read the graph instantly rather than load the graph at start-up as other
libraries would.  This makes deploying code changes quick and easy.
It requires only 8 bytes per edge for bi-directional edge access, so, for example, it can store the
[Twitter-2010](http://law.di.unimi.it/webdata/twitter-2010/) graph of 1.5 billion edges using 
12GB of RAM.  We use the Tempest library in production, and it has careful unit tests.

### Created by
Tempest was built by a team of Stanford PhDs who have built state-of-the-art systems and algorithms for large scale network analytics at Stanford, Twitter and Amazon. If you have any questions about using Tempest or have feature requests, create a Github issue or 
send an email to <peter.lofgren@cs.stanford.edu>.


## Using Tempest DB (alpha)
1. Tempest's dependencies (including postgres and java) are neatly packaged in a Docker container.  To
use Tempest DB, install docker on your machine, then run
   
   `docker run -t -i -v ~/:/mnt/home/ -p 127.0.0.1:10001:10001 teapotco/tempestdb:v0.12.0 bash`
   (Mounting your home directory using `-v ~/:/mnt/home/` is optional, but could be useful for reading
   your graph files in docker.)  Inside docker, if you've cloned tempest on the host file system, 
   symlink `~/tempest` to the location under `~/mnt/home/` where tempest is cloned.  Otherwise
   inside docker run `cd; git clone https://github.com/teapot-co/tempest.git`.
2. You need three files to fire up the Tempest server with your graph: 
      a) a data file for nodes,
      b) a data file for edges, and
      c) a config file.

  Tempest expects a node file and an edge file in a csv format without headers. To see the format of 
  the node file and the edge file expected by Tempest, look at `/root/tempest/example/example_nodes.csv`
  and `/root/tempest/example/example_edges.csv`.
  
3. Once you have your nodes and edges in this format, copy the config file `/root/tempest/example/example_graph.yaml`
   to a new file, for example `/root/users_graph.yaml`.   The following fields should be changed:
      a) graph_name: This is name of your new database. E.g. "users". Database and attribute names may contain alphanumeric characers and '_', and must start with an alphabet character (a-z).
      b) node_file: for example "/mnt/home/data/users.csv"
      c) node_attributes: Enter the name and type of each attribute in your node_file.csv. Attributes type may only be 'string', 'int' (32 bit), 'bigint' (64 bit), or 'boolean'. Enter the attributes in the same order as they appear in the node file.
      d) edge_file: for example "/mnt/home/data/edges.csv"
      e) node_identifier_field: This is the name of the attribute in the node file that is used to identify edges in the edge file. For example, if your edge file looks like:
        > alice,bob
        > bob,carol
        where, 'alice', 'bob', etc. are usernames from the node file, then the node_identifier_field should be set to 'username'.
      f) database_caching_ram: This is RAM used by Postgres to cache node attributes.  Additional RAM is used by Tempest to store the edges in RAM. Set it to roughly 1/4 of your system's RAM.
   
4. Convert your graph to binary and load your nodes and edges into Postgres by running
   
   `create_graph.sh <your new config file>`
   
   Depending on the size of your initial graph, this step may take up to a few hours. For example, for a graph of 1B edges, this step will take about 4 hours. We realize that is a long time to wait to get your hands on Tempest, but this is one-time hassle: once Tempest is initialized, stopping/starting only takes a few seconds.

5. Start the server
   
   `start_server.sh /root/tempest/config/graph.yaml`
   
6. Now you can connect to the server from inside or outside docker.  If outside docker, first run 
   `pip install tempest_db`. Then in `ipython` you can run, for example
   ```
   import tempest_db
   c = tempest_db.TempestClient()
   alice_id = c.nodes("example", "username = 'alice'")[0]
   alice_neighbors = c.out_neighbors(alice_id)
   alice_neighbor_names = c.multi_node_attribute("example", alice_neighbors, "name")
   ```

### Recommended Machine Sizes

|Number of edges in the graph | EC2 Instance type | Volume size (in GB) |
|-----------------------------|-------------------|--------------------|
| < 50M    | r3.large   | 100   |
| 50M-500M | r3.xlarge  | 200   |
| 500M-5B  | r3.2xlarge | 2000  |
| 5B-10B   | r3.4xlarge | 5000  |
| 10B+     | r3.8xlarge | 10,000|

## Using Tempest as a library

### Requirements
Tempest depends on Java and [sbt](http://www.scala-sbt.org/download.html).

### Converting a Graph to Tempest Format
For large graphs, you will want to first convert your graph to the Tempest binary format.  The 
input to our converter is a simple text format where each line has the form "id1 id2" to 
indicate an edge from id1 to id2, where id1 and id2 are integers between 0 and 2^31.  To convert 
the test graph, for example, run:
```
git clone https://github.com/teapot-co/tempest.git
cd tempest
bin/convert_graph_immutable.sh src/test/resources/test_graph.txt test_graph.dat
```
### Languages Supported by Tempest
#### Scala 
Simply add
```
libraryDependencies += "co.teapot" %% "tempest" % "0.12.0"
```
to your `build.sbt` file.  The graph classes in tempest are the following:

- [DirectedGraph](http://teapot-co.github.io/tempest/scaladoc/#co.teapot.tempest.graph.DirectedGraph)
  defines the interface for graphs
- [MemoryMappedDirectedGraph](http://teapot-co.github.io/tempest/scaladoc/#co.teapot.tempest.graph.MemoryMappedDirectedGraph)
  efficiently reads an immutable graph from a Tempest binary graph file
- [MemoryMappedMutableDirectedGraph](http://teapot-co.github.io/tempest/scaladoc/#co.teapot.tempest.graph.MemMappedDynamicDirectedGraph)
  efficiently stores a large graph in a binary file and allows efficient edge additions
- [ConcurrentHashMapDynamicGraph](http://teapot-co.github.io/tempest/scaladoc/#co.teapot.tempest.graph.ConcurrentHashMapDynamicGraph)
  is a simple mutable graph class based on ConcurrentHashMap
    
As an example of using the graph methods, convert the test graph to test_graph.dat as described 
above, then run the following from the tempest directory:
```
sbt console
val graph = co.teapot.tempest.graph.MemoryMappedDirectedGraph("test_graph.dat")
graph.nodeCount
graph.edgeCount
graph.outDegree(4)
graph.outNeighbors(4)
graph.inNeighbors(4)
```

#### Java
Because scala compiles to the jvm, you can naturally use Tempest from Java.  Tempest is in a maven repo,
so you're using maven, simply add
```
<dependency>
      <groupId>co.teapot</groupId>
      <artifactId>tempest_2.11</artifactId>
      <version>0.12.0</version>
    </dependency>
```
to your pom.xml file to access the Tempest dependency.  Then for example, in Java you can 
`import co.teapot.tempest.graph.MemoryMappedDirectedGraph` and write
```
MemoryMappedDirectedGraph graph = new MemoryMappedDirectedGraph(new File("test_graph.dat"));
System.out.println("out-neighbors of 2: " + graph.outNeighborList(2));
```
The documentation javadoc for the graph classes is linked above in the Scala section.

#### Python
To use tempest from python, it has a client/server mode, where the server
stores large graphs in RAM, while the client is conveniently in python.  The server and client can even
be on different machines; for example the client can run on a web server while the server runs in a large
graph server.

First clone the repository and convert the graph to Tempest binary format, as described
above.  Then start the server, supplying the binary graph name, for example
```
bin/start_graph_only_server.sh test_graph.dat 
```
The server runs by default on TCP port 10001, but you can change this, for example
```
bin/start_server.sh test_graph.dat 12345
```

Finally,  connect to the server from python.  First run `pip install tempest_db`.  Then
from the python console:
```
>>> import tempest_db
>>> graph = tempest_db.client()
>>> graph.max_node_id()
9
>>> graph.out_degree(2)
4
>>> graph.out_neighbors(2)
[0, 1, 3, 9]
```

## Project Roadmap
- Improve support for multiple edge types.
- Support edge attributes.
