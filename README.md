# Pallet-Hadoop

This project serves as an example to get you started using [Pallet-Hadoop](https://github.com/pallet/pallet-hadoop), a layer over [Pallet](https://github.com/pallet/pallet) that translates data descriptions of Hadoop clusters into fully configured, running machines. For a more detailed discussion of Pallet-Hadoop's design, see the [project wiki](https://github.com/pallet/pallet-hadoop/wiki).


## Setting Up

Before you get your first cluster running, you'll need to [create an AWS account](https://aws-portal.amazon.com/gp/aws/developer/registration/index.html). Once you've done this, navigate to [your account page](http://aws.amazon.com/account/) and follow the "Security Credentials" link. Under "Access Credentials", you should see a tab called "Access Keys". Note down your Access Key ID and Secret Access Key for future reference.

I'm going to assume that you have some basic knowledge of clojure, and know how to get a project running using [leiningen](https://github.com/technomancy/leiningen) or [cake](https://github.com/ninjudd/cake). Go ahead and download [the example project](https://github.com/pallet/pallet-hadoop-example) to follow along:


```bash
$ git clone git://github.com/pallet/pallet-hadoop-example.git
$ cd pallet-hadoop-example
```

Open up `./src/pallet-hadoop-example/core.clj` with your favorite text
editor. `make-example-cluster` is a function that builds a data description of a full hadoop cluster with:

* One master node functioning as jobtracker and namenode
* A number of slave nodes (`slave-count`), each acting as datanode and
  tasktracker.
  
For convenience, I've bound `example-cluster` to a cluster definition with 2 slaves and a jobtracker/namenode master node.

Start a repl:

```clojure
$ lein repl
user=> (use 'pallet-hadoop-example.core) (bootstrap)
```

### Compute Service

Pallet abstracts away details about specific cloud providers through the idea of a "compute service". The combination of our cluster definition and our compute service will be enough to get our cluster running. A compute service is defined at the REPL like so:

```
user=> (def ec2-service
           (compute-service "aws-ec2"
                            :identity "ec2-access-key-id"         ;; Swap in your access key ID
                            :credential "ec2-secret-access-key")) ;; Swap in your secret key
#'user/ec2-service
```

Alternatively, if you want to keep these out of your code base, save the following to `~/.pallet/config.clj`:

```clojure
(defpallet
  :services {:aws {:provider "aws-ec2"
                   :identity "your-ec2-access-key-id"
                   :credential "your-ec2-secret-access-key"}})
```

and define `ec2-service` with:

```clojure
 user=> (def ec2-service (compute-service-from-config-file :aws))
#'user/ec2-service
```

### Booting the Cluster

Now that we have our compute service and our cluster defined, booting the cluster is as simple as the following:

```clojure
user=> (create-cluster example-cluster ec2-service)
```

The logs you see flying by are Pallet's SSH communications with the nodes in the cluster. On node startup, Pallet uses your local SSH key to gain passwordless access to each node, and coordinates all configuration using streams of SSH commands.

Once `create-cluster` returns, we're done! We now have a fully configured, multi-node Hadoop cluster at our disposal.

### Running Word Count

To test our new cluster, we're going log in and run a word counting MapReduce job on a number of books from [Project Gutenberg](http://www.gutenberg.org/wiki/Main_Page).

At the REPL, type

```clojure
user=> (jobtracker-ip :public ec2-service)
```

This will print out the IP address of the jobtracker node. I'll refer to this address as `jobtracker.com`.

Point your browser to `jobtracker.com:50030`, and you'll see the JobTracker web console. (Keep this open, as it will allow us to watch our MapReduce job in action.).`jobtracker.com:50070` points to the NameNode console, with information about HDFS.

The next step is SSH into the jobtracker and operate as the hadoop user. Head to your terminal and run the following commands:

```bash
$ ssh jobtracker.com (insert actual address, enter yes to continue connecting)
$ sudo su - hadoop
```

### Copy Data to HDFS

At this point, we're ready to begin following along with Michael Noll's excellent [Hadoop configuration tutorial](http://goo.gl/aALr9). (I'll cover some of the same ground for clarity.)

Our first step will be to collect a bunch of text to process. Start by downloading the following seven books to a temp directory:

* [The Outline of Science, Vol. 1 (of 4) by J. Arthur Thomson](http://www.gutenberg.org/cache/epub/20417/pg20417.txt)
* [The Notebooks of Leonardo Da Vinci](http://www.gutenberg.org/cache/epub/5000/pg5000.txt)
* [Ulysses by James Joyce](http://www.gutenberg.org/cache/epub/4300/pg4300.txt)
* [The Art of War by 6th cent. B.C. Sunzi](http://www.gutenberg.org/cache/epub/132/pg132.txt)
* [The Adventures of Sherlock Holmes by Sir Arthur Conan Doyle](http://www.gutenberg.org/cache/epub/1661/pg1661.txt)
* [The Devil’s Dictionary by Ambrose Bierce](http://www.gutenberg.org/cache/epub/972/pg972.txt)
* [Encyclopaedia Britannica, 11th Edition, Volume 4, Part 3](http://www.gutenberg.org/cache/epub/19699/pg19699.txt)

For convenience, pallet-hadoop-example has created a script that will download all such files for you. Running the following commands at the remote shell should do the trick. The books will be downloaded into `/tmp/books`:

```bash
$ cd /tmp
$ ./download-books.sh
```

Next, navigate to the Hadoop directory:

```bash
$ cd /usr/local/hadoop-0.20.2/
```

And copy the books over to the distributed filesystem:

```bash
/usr/local/hadoop-0.20.2$ hadoop dfs -copyFromLocal /tmp/books books
/usr/local/hadoop-0.20.2$ hadoop dfs -ls
Found 1 items
drwxr-xr-x   - hadoop supergroup          0 2011-06-01 06:12:21 /user/hadoop/books
/usr/local/hadoop-0.20.2$ 
```

### Running MapReduce

It's time to run the MapReduce job. `wordcount` takes an input path within HDFS, processes all items within, and saves the output to HDFS -- to `books-output`, in this case. Run this command:

```bash
/usr/local/hadoop-0.20.2$ hadoop jar hadoop-examples-0.20.2-cdh3u0.jar wordcount books/ books-output/
```

And you should see something very similar to this:

```bash
11/06/01 06:14:30 INFO input.FileInputFormat: Total input paths to process : 7
11/06/01 06:14:30 INFO mapred.JobClient: Running job: job_201106010554_0002
11/06/01 06:14:31 INFO mapred.JobClient:  map 0% reduce 0%
11/06/01 06:14:44 INFO mapred.JobClient:  map 57% reduce 0%
11/06/01 06:14:45 INFO mapred.JobClient:  map 71% reduce 0%
11/06/01 06:14:46 INFO mapred.JobClient:  map 85% reduce 0%
11/06/01 06:14:48 INFO mapred.JobClient:  map 100% reduce 0%
11/06/01 06:14:57 INFO mapred.JobClient:  map 100% reduce 33%
11/06/01 06:15:00 INFO mapred.JobClient:  map 100% reduce 66%
11/06/01 06:15:01 INFO mapred.JobClient:  map 100% reduce 100%
11/06/01 06:15:02 INFO mapred.JobClient: Job complete: job_201106010554_0002
11/06/01 06:15:02 INFO mapred.JobClient: Counters: 22
11/06/01 06:15:02 INFO mapred.JobClient:   Job Counters 
11/06/01 06:15:02 INFO mapred.JobClient:     Launched reduce tasks=3
11/06/01 06:15:02 INFO mapred.JobClient:     SLOTS_MILLIS_MAPS=74992
11/06/01 06:15:02 INFO mapred.JobClient:     Total time spent by all reduces waiting after reserving slots (ms)=0
11/06/01 06:15:02 INFO mapred.JobClient:     Total time spent by all maps waiting after reserving slots (ms)=0
11/06/01 06:15:02 INFO mapred.JobClient:     Launched map tasks=7
11/06/01 06:15:02 INFO mapred.JobClient:     Data-local map tasks=7
11/06/01 06:15:02 INFO mapred.JobClient:     SLOTS_MILLIS_REDUCES=46600
11/06/01 06:15:02 INFO mapred.JobClient:   FileSystemCounters
11/06/01 06:15:02 INFO mapred.JobClient:     FILE_BYTES_READ=1610042
11/06/01 06:15:02 INFO mapred.JobClient:     HDFS_BYTES_READ=6557336
11/06/01 06:15:02 INFO mapred.JobClient:     FILE_BYTES_WRITTEN=2753014
11/06/01 06:15:02 INFO mapred.JobClient:     HDFS_BYTES_WRITTEN=1334919
11/06/01 06:15:02 INFO mapred.JobClient:   Map-Reduce Framework
11/06/01 06:15:02 INFO mapred.JobClient:     Reduce input groups=121791
11/06/01 06:15:02 INFO mapred.JobClient:     Combine output records=183601
11/06/01 06:15:02 INFO mapred.JobClient:     Map input records=127602
11/06/01 06:15:02 INFO mapred.JobClient:     Reduce shuffle bytes=958780
11/06/01 06:15:02 INFO mapred.JobClient:     Reduce output records=121791
11/06/01 06:15:02 INFO mapred.JobClient:     Spilled Records=473035
11/06/01 06:15:02 INFO mapred.JobClient:     Map output bytes=10812590
11/06/01 06:15:02 INFO mapred.JobClient:     Combine input records=1111905
11/06/01 06:15:02 INFO mapred.JobClient:     Map output records=1111905
11/06/01 06:15:02 INFO mapred.JobClient:     SPLIT_RAW_BYTES=931
11/06/01 06:15:02 INFO mapred.JobClient:     Reduce input records=183601
/usr/local/hadoop-0.20.2$ 
```

### Retrieving Output

Now that the MapReduce job has completed successfully, all that remains is to extract the results from HDFS and take a look.

```bash
$ mkdir /tmp/books-output
$ hadoop dfs -getmerge books-output /tmp/books-output
$ head /tmp/books-output/books-output
```

You should see something very close to:

```text
"'Ah!'	2
"'Ample.'	1
"'At	1
"'But,	1
"'But,'	1
"'Come!	1
"'December	1
"'For	1
"'Hampshire.	1
"'Have	1
```

Success!

### Killing the Cluster

When you're finished, you can kill the cluster with this command, back at the REPL:

```clojure
user=> (destroy-cluster example-cluster ec2-service)
```
