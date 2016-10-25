---
layout: post
title:  "HBase: My Study Note"
date: 2016-10-23 06:25:06 -0700
comments: true
---

HBase: My Study Note
====================

I've never used <a href="http://hbase.apache.org/">Apache HBase</a> before, and now would be a great time to 
give it a try. I am also pretty new to Docker, so I figured, why not try them together?

&nbsp;

#### Running HBase (via Docker)

```
> docker pull oddpoet/hbase-cdh5
> docker run -p 2181:2181 \
  -p 65000:65000 \
  -p 65010:65010 \
  -p 65020:65020 \
  -p 65030:65030 \
  -h $(hostname) -d oddpoet/hbase-cdh5
```

Now if we load our browser at `http://localhost:65010/` and `http://localhost:65030/` we should see the 
master & region server status.


&nbsp;

#### Shell Access

Next step is to use the interactive shell:

```
> docker ps

CONTAINER ID        IMAGE                COMMAND
5a0ca2bf030e        oddpoet/hbase-cdh5   "/bin/bash start.sh"
...
```

Mark the "CONTAINER ID" value, and issue a command to attach an interactive shell into the running container:

```
> docker exec -i -t 5a0ca2bf030e /bin/bash

[root@Nings-MacBook-Pro /]#
```

Now we are inside the container that is running the HBase instance. If we run the command `hbase shell` now 
we can interact with the HBase. The following session creates a table, add some data, and then display and 
query them:

(<b>Please note that the following commands can be pretty slow, almost like they are failing</b>)

```
[root@Nings-MacBook-Pro /]# hbase shell

hbase(main):001:0> create 'testTable', 'cf'
0 row(s) in 15.8690 seconds
=> Hbase::Table - testTable

hbase(main):002:0> put 'testTable', 'r1', 'cf:c1', 'v1'
0 row(s) in 0.1400 seconds

hbase(main):003:0> put 'testTable', 'r2', 'cf:c1', 'v2'
0 row(s) in 0.0180 seconds

hbase(main):004:0> scan 'testTable'
ROW                                  COLUMN+CELL
 r1                                  column=cf:c1, timestamp=1477295738212, value=v1
 r2                                  column=cf:c1, timestamp=1477295800618, value=v2
2 row(s) in 0.0320 seconds

hbase(main):005:0> get 'testTable', 'r2'
COLUMN                               CELL
 cf:c1                               timestamp=1477295800618, value=v2
1 row(s) in 0.0160 seconds
```

&nbsp;

#### Java Client

My goto language is Java, so this study is not complete without a very simple working Java client.

We will be using the official hbase-client library:

```
(pom.xml)
    <dependencies>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.2.3</version>
        </dependency>
    </dependencies>
```

and the Java code:

```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

public class TestApp {

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Configuration config = HBaseConfiguration.create();
        Connection connection = ConnectionFactory.createConnection(config);
        Table table = connection.getTable(TableName.valueOf("testTable"));

        Get g = new Get(Bytes.toBytes("r2"));
        Result r = table.get(g);
        byte [] value = r.getValue(Bytes.toBytes("cf"), Bytes.toBytes("c1"));
        String valueStr = Bytes.toString(value);
        System.out.println("GET: " + valueStr);

        table.close();
    }
}
```
And when we run this java code we see the result "v2", which was what we added earlier:
`r2 column=cf:c1, timestamp=1477295800618, value=v2`
So it works!

<br/><br/><br/>
<br/><br/><br/>