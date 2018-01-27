---
title: "Spark and the Minor Planet Center data part 3"
date: 2017-12-05T19:39:29Z
draft: false
slug: "spark_and_mpc_part_3"
tags: ["spark", "minor planet center", "asteroids"]
image: "img/fire3.jpg" 
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
---

In the [last post](/post/spark_and_mpc_part_2) we read the minor planet center observation file. This was a fixed width text file. We only pulled a couple of columns out of it, but we learnt to use User Defined Functions, groupBy and select. In the [first post](/post/spark_and_mpc) of this series we covered reading a json file which contained information about all the asteroids we know about.

This time we are going to join the two data sets together and finally solve our original problem, which was to find the full date of the earliest observation of each un-numbered object. The json file we first looked at contained the year but not days and months. The observation file has the full date down to smaller than minutes accuracy, but it does not have the orbital parameters.

So we need to join the two files up to get our results.

# Assumptions

You have been through the previous posts in this series and understood them. You have a project already set up and able to read and process the orbit and observation files.

# Setup

As you should already have the project and the two required data files you are already set up.

# Development

Thankfully Spark SQL makes this bit really easy. the `.join()` function on a data set allows you do all the kinds of joins you could do in a database. For this we need a simple inner join, which is the default option so you won't see us having to define it in the code below.

```scala
val joined = orbRec.join(obs, Seq("id", "id"))
```

There is not a lot of difference in which way around you do this from the results point of view. There may be differences in performance but that will depend on your data set. In this case with the minor planets data it doesn't really matter which way around we do it. If you now add a `joined.show(2)` line you will be able to see the columns from both data sets are now next to each other.

There are several ways to tell Spark SQL that you want to match the two id columns up. `Seq("id", "id")` is probably the simplest but you can also do something like `obs.col("id").equalTo(orbRec.col("id"))` This is really useful if you want to use something that is not a simple equals when joining.

Now we don't really want all those columns in the output so we can add a `.select()` call to clean things up a bit.

```scala
  .select(
    obs.col("id"),
    col("Name"),
    col("date"),
    col("a"),
    col("e"),
    col("i"),
    col("Epoch"),
    col("H"),
    col("G"),
    col("Node"),
    col("Peri")
  )
```

Unfortunately for me at this point things went bang. I got an exception about the query plan being too big. My first attempt at fixing this was adding a `.select()` call to the orbRec processing chain to trim down the number of columns. This didn't work. In fact I think it made things worse. The query plan is now longer.

The solution is to add checkpoints. Checkpoints allow spark sql to split the query plan in to bits. The checkpoints are written to disk and persistent. This is also useful if you need to reuse a data frame multiple times but it is expensive to compute.

The first thing we need to do is define where to put the checkpoint files. Just under where we create the `SparkSession` we need to add

```scala
spark.sparkContext.setCheckpointDir("/tmp/")
```

This will cause spark to store its checkpoint data in `/tmp` you may want to put it somewhere different depending on what you are running on. Now we just need to tell spark where to checkpoint. This is done with the `.checkpoint()` function. I tend to find it best to checkpoint before both sides of a join and any time you are going to reuse a data frame.

We simply need to add the call to `.checkpoint()` at the end of the two chains we defined for `orbRec` and `obs`. Now when we run `joined.show(2)` we should get something like

```text
+---------+----+-------------------+---------+---------+-------+---------+----+----+---------+---------+
|       id|Name|               date|        a|        e|      i|    Epoch|   H|   G|     Node|     Peri|
+---------+----+-------------------+---------+---------+-------+---------+----+----+---------+---------+
|1995 SR42|null|1995-09-20T09:21:27|2.3527416|0.1811627|2.01897|2449980.5|19.0|0.15|117.61112|171.74016|
|  1996 LW|null|1996-06-09T07:21:53|2.1578017|0.2848271|7.64274|2450240.5|19.5|0.15|194.61088| 128.2009|
+---------+----+-------------------+---------+---------+-------+---------+----+----+---------+---------+
only showing top 2 rows
```

Now the only thing left to do is write the data back out to disk. Here we will run into one of the interesting design problems of running spark locally.

Saving data as a csv file is as easy as reading a file. You just need a `write` rather than a `read`

```scala
  joined.write.mode(SaveMode.Overwrite).csv("output")
```

Note the lack of file extension on the target name. This is because `output` will be a folder and not a file. Under the hood spark will split the data frame into partitions. These are basically groups of rows that will all be on the same machine in a clustered environment. In our environment every thing is on the one machine, however in a cluster you would get one `output` directory on each machine with the partitions that were on that machine written out.

After running the program, inside the `output` folder you will find a large number of files. There will be a `_SUCCESS` file created when every thing has finished. There will be one `part-*.csv` file and one `.part-*.csv.crc` file per partition in your data. The file ending in .crc is a checksum allowing you to verify that the data has written correctly if you need it. The files ending in .csv will be all the data you asked Spark SQL to write out.

The last step is to join every thing up. I find this easiest to do from the command line. (I'm on a linux machine with bash)

```scala
cat output/part-*.csv > output.csv
```

This will generate an output.csv file which is all the parts joined together. Note that you will not be able to control the order of the rows using this.

And there we go. We now have the orbital elements for all the un-numbered objects with the date of their first observation. In this part we have learnt how to join to data frames together and how to output data. This part has been a bit shorter than the others but joins every thing together. (Pun intended)

You can find all the code for this [project on my github](https://github.com/wselwood/orbitdates)

I hope this is useful to you. If you have any questions please let me know on twitter.