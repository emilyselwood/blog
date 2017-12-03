---
title: "Spark and the Minor Planets Center data"
date: 2017-12-02T08:55:24Z
draft: false
slug: "spark_and_mpc"
tags: ["spark", "minor planets center", "asteroids"]
image: "img/fire.jpg"
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
authoravatar: "img/profilePic.jpg"
---
# Introduction

A few weeks ago I saw comments between [@Sondy](https://twitter.com/sondy) and [@JLGalache](https://twitter.com/JLGalache) talking about getting a list of asteroids with their date of discovery. The main data file lists the year of discovery but not the actual date. I thought there was a way to get this information by looking at the observation file and joining it to the main data file. Todo this I decided to use Apache Spark. In this post I'll go through setting up the spark environment and reading the json object file.

## What is spark?

[Spark](https://spark.apache.org/) is an in-memory distributed processing framework. It is one of the largest open source data processing frameworks. There is a core section and several modules built on top. For this we will be using Spark SQL.

## What is the Minor Planets Center?

The [minor planets center](http://www.minorplanetcenter.net/iau/mpc.html) is an organization that keeps track of observations of asteroids. They keep a list of all the known asteroids and all the observations people around the world have made. They publish this data in a range of formats from a fixed width text file to json. Their documentation has improved a lot in the last couple of years.

## What data are we looking at?

A JSON file which contains information about all the known asteroids. This has their orbital parameters, id, name, discovery year, etc. Don't worry too much we won't get in to orbital mechanics here. (Mostly because I struggle with it my self)

While these data files are not big data by any means they are real data that has actual quirks and is not tiny which makes them a good thing to practice with. They shouldn't take hours to process.

# Assumptions

I am going to assume you have a local java installation and an IDE you are comfortable with. You know some scala. You can probably work out whats going on but it will be a lot easier if you are aware of the scala syntax before you start reading this.

# Setup

First we need to go and get the data files. These are reasonably big. On the minor planets center [data page](http://www.minorplanetcenter.net/data) there is a link to the `mpcorb_extended.json.gz` file. Download this. This might take a while. So feel free to get started with the next bit.

I use [intelliJ](https://www.jetbrains.com/idea/). So create a new [gradle](https://gradle.org/) project. There is no particular reason I chose Gradle over SBT or maven only I have more experience with it. The basics of the project setup are easy enough. First add a few bits to mark the project as a scala project and create variables for versions of things. This will save repeating your self if you need to change them in the future. The versions I used were simply the latest version of spark and its matching scala version when I did this. Open up the build.gradle file.

```groovy
apply plugin : 'scala' // Adding the scala flag adds steps to the build process for the scala compiler

project.ext.scalaVersion = '2.11.9'
project.ext.sparkVersion = '2.2.0'

sourceCompatibility = 1.8 // Use Java 1.8 compatibility. Mostly this is a hint for the IDE rather than the build.
```

Then add the following dependencies to the dependencies section of your gradle file, there should be a junit dependency there already.

```groovy
    compile "org.scala-lang:scala-library:${project.ext.scalaVersion}"
    compile "org.scala-lang:scala-reflect:${project.ext.scalaVersion}"
    compile "org.scala-lang:scala-compiler:${project.ext.scalaVersion}"
    compile "org.apache.spark:spark-core_2.11:${project.ext.sparkVersion}"
    compile "org.apache.spark:spark-sql_2.11:${project.ext.sparkVersion}"
```

# Development

Now we need some actual code. Create a file in the `src/main/scala/<Your package here>` directory called something like `orbitdates.scala`

Create a main method:

```scala
object OrbitDates {
  def main(args: Array[String]) : Unit = {
    // Our code will go here
  }
}
```

Note: with spark you must create a main method this way and not use the `extends App` style syntax because the objects will be serialised and sent to worker nodes by spark later on. You will get very strange errors due to the order that the generated main method does things.

It is probably worth adding a `println("hello world")` at this point and making sure the project builds and runs. `./gradlew build` from the command line or the gradle build button in your ide should run cleanly. There will probably be a lot of downloading to be done the first time.

The first step of a spark program is getting hold of the right kind of spark context or session.

```scala
    val spark = SparkSession
      .builder()
      .master("local[2]") // if you have more cores or a cluster turn this up.
      .appName("Spark SQL join observations")
      .getOrCreate()
```

As we want to use Spark SQL we need a spark session. This needs to be told where our spark "cluster" is. For this we are just using local with two threads. If you have a more powerful machine feel free to turn up the number of threads. If you are lucky enough to have an access to a cluster you should put the address of the "master" node here. (Note: the master/slave terminology that spark uses is horrible. Controller/worker would make more sense and be less offensive)

The `.appName()` is just a name for your app. It can be anything. If you are running in a cluster this is what shows up on the web front end.

The `.getOrCreate()` call at the end of the chain returns the spark context you have just setup. If for some reason your code already has an active spark context this will handle that as well.

To make that work you will also need to include

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.catalyst.expressions.Substring
import org.apache.spark.sql.functions._
```

The first one covers the `SparkSession`. The other two we will get to later.

To make things clearer later I added a mapping from the arguments simply so I didn't have to remember if the data file was args(0) or args(34)

```scala
val mpcobs = args(0) // The json data file of objects
```

Now we have the spark session and something holding the path to our input files we can start trying to read them. First lets start with the json data file.

```scala
val orbRec = spark.read
      .option("multiLine", true)
      .json(mpcobs)
```

This will give us what is called a data frame in spark sql parlance. Basically its a lazy representation of the data file. Nothing will actually be done until we ask for something to be returned. `spark.read` contains methods for reading lots of different kinds of data sources. These data sources all have different options that can be applied to them. Here we ask to read a multi line json file. This option has only existed since spark 2.2. We need the multi line option due to the way the MPC json file is layed out.

This will give us access to the data for each object as a record. To see how the data looks you can use the `.show()` function. This will display a simple table of the data to stdout when the program is run. This is very useful for working out how your data looks. There is also `.describe("column name")` function which will give you statistical information about your column.

The next step is to sort out the number column and remove the brackets that are around it. After that we can find the correct column to use for the id.

```scala
  .withColumn("number_formatted", new Column(Substring(col("Number").expr, lit(2).expr, (length(col("Number"))-2).expr)))
  .withColumn("id", coalesce(col("number_formatted"), col("Principal_desig")))
```

The first line adds an extra column that does a substring of the original column "Number" removing the first and last two characters from the column.
The second line creates a new column called "id" which will contain the "number_formatted" column if it is not null or it will use the "Principal_desig" column. The `coalesce` function is very useful when you need to pick the first non null column from a list of options.

Now you should have a block that looks like this:

```scala
  val orbRec = spark.read
    .option("multiLine", true)
    .json(mpcobs)
    .withColumn("number_formatted", new Column(Substring(col("Number").expr, lit(2).expr, (length(col("Number"))-2).expr)))
    .withColumn("id", coalesce(col("number_formatted"), col("Principal_desig")))
```

This simple code will read in the file and create an "id" column. If you add a `orbRec.show(2)` on the end you should see a very long table like below. There will also be a load of logging of what spark is doing. It should end up in stderr rather than stdout. You can see the last two columns are the ones we added.

```table
+-------------+----------+---------+--------+----------------------------------+---------+----+----+---------+----------+---------+--------+------+---------+-------+--------+------+---------------+--------------------------+----------+--------------+------------+--------+---------+---------------+----------+------------+---------------+---------+----------------+--------------+-------------+---+---------+---------+--------+----------+----+----------------+---+
|Aphelion_dist|Arc_length|Arc_years|Computer|Critical_list_numbered_object_flag|    Epoch|   G|   H|Hex_flags|  Last_obs|        M|NEO_flag|  Name|     Node|Num_obs|Num_opps|Number|One_km_NEO_flag|One_opposition_object_flag|Orbit_type|Orbital_period|Other_desigs|PHA_flag|     Peri|Perihelion_dist|Perturbers|Perturbers_2|Principal_desig|      Ref|Semilatus_rectum|Synodic_period|           Tp|  U|        a|        e|       i|         n| rms|number_formatted| id|
+-------------+----------+---------+--------+----------------------------------+---------+----+----+---------+----------+---------+--------+------+---------+-------+--------+------+---------------+--------------------------+----------+--------------+------------+--------+---------+---------------+----------+------------+---------------+---------+----------------+--------------+-------------+---+---------+---------+--------+----------+----+----------------+---+
|     2.976646|      null|1801-2017|MPCLINUX|                              null|2458000.5|0.12|3.34|     0000|2017-03-05|309.49412|    null| Ceres| 80.30888|   6672|     113|   (1)|           null|                      null|       MBA|     4.6037329|   [1943 XB]|    null| 73.02368|      2.5581728|       M-v|         30h|        A899 OF|MPO399990|       1.3757948|       1.27749|2458236.41089|  0|2.7674094|0.0756074|10.59322|0.21408881| 0.6|               1|  1|
|    3.4125514|      null|1821-2017|MPCLINUX|                              null|2458000.5|0.11|4.13|     0000|2017-10-05|291.65136|    null|Pallas|173.08718|   7910|     108|   (2)|           null|                      null|       MBA|     4.6179031|        null|    null|309.99154|       2.133619|       M-v|         28h|           null|MPO421624|        1.312813|     1.2764032|2458320.73644|  0|2.7730852|0.2305974|34.83792|0.21343186|0.58|               2|  2|
+-------------+----------+---------+--------+----------------------------------+---------+----+----+---------+----------+---------+--------+------+---------+-------+--------+------+---------------+--------------------------+----------+--------------+------------+--------+---------+---------------+----------+------------+---------------+---------+----------------+--------------+-------------+---+---------+---------+--------+----------+----+----------------+---+
only showing top 2 rows

```

If you get errors about array index out of bounds you may have forgotten to pass the parameter for where the data file is.

Now you can start to explore a bit. Find out the stats of the columns using the `.describe()` function. Play with the `.select()` function to limit the number of columns. The `.where()` function can be used to filter the data. You can use `.agg()` with `min()` and`max()` to aggregate data. The `.count()` function will return the number of rows in a data set.

## Challenge

* How many asteroids were first observed in the year 2000?

The first person to send me a tweet with the answer will get a virtual hi-five and some kudos.

# Conclusion

Here we are going to leave it for today. I'm planning to do another couple of posts. One reading the observation data file and another on joining every thing up. I hope this is useful and you found it interesting. If you did or you have questions please let me know on twitter.

# Thanks

Thanks to the Minor planet center for making this data freely available.

Thanks to Sondy and JL for the idea.