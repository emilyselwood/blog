---
title: "Spark and the Minor Planet Center data part 2"
date: 2017-12-03T15:39:22Z
author: "Emily Selwood"
draft: false
slug: "spark_and_mpc_part_2"
tags: ["spark", "minor planet center", "asteroids"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

In the [last post](/post/spark_and_mpc) we read the minor planet center orbit file. This was a JSON text file. This time we are going to look at a bit more complex file to process. If you haven't read the first post in this series I recommend starting there before reading this.

In this post we are going to be looking at the Observation file. There are two parts to this file. One is for the numbered objects and other other for the un-numbered objects. Due to the original idea for this project we are going to work with the un-numbered file today. 

# Assumptions

You have been through the previous post in this series and understood it. You have a project already set up and able to read and process the orbit file.

# Setup

The data file we are after can be found on the [mpcat-obs page](http://www.minorplanetcenter.net/iau/ECS/MPCAT-OBS/MPCAT-OBS.html) under the link for the un-numbered minor planets. Download it. Then while it is downloading open up the project you created following along from the last post.

# Development

First lets add another variable so we don't have to remember which argument is which.

```scala
val observationPath = args(1)
```

Now to get on to reading the file. In some ways this one is easier and some ways more complex. The data we are looking at here is a fixed width format. This means that there is nothing to separate the columns just that they always start in the same place no matter how long the data contained in them gets.

Unfortunately there is not a built in function to deal with this easily. So we will have to read the file line by line and then split it up our selves. Actually we will cheat a little bit and only pull out the bits we need so as not to waste time doing work we don't need to do.

```scala
  val obs = spark.read.text(observationPath)
```

Like with the JSON reader from the previous post it is very easy to read a text file in spark sql. This time we don't even need any options. This will create a data frame which contains each line from the file in a column called "value"

The first step is always to check the data looks how you expect it to look. Use the `.show()` function to have a look at the first few rows and get a feel for how the data looks. To make our lives easier however there is some [documentation available](http://www.minorplanetcenter.net/iau/info/ObsFormat.html) The column formats are defined in fortran format, I am pretty sure this is because that is what is used to generate the files. However it's a pretty simple format.

A Means ascii and then the number following it is the number of characters allowed. e.g

```text
Columns     Format   Use
    1 -  5       A5     Minor planet number
    6 - 12       A7     Provisional or temporary designation
   13            A1     Discovery asterisk
```

The columns are numbered from 1 rather than 0 as an array would be. Which is another thing to remember. The first thing we need to do is extract an id column. Similar to what we did with the json file. In a future post we will use this to join the two files together.

We can use the substring function to do this. Because we are using the un-numbered file we need to be looking for the provisional to temporary designation column for the ID. We could use both and coalesce but as we know what data we are putting in we might as well keep things simple.

```scala
  .withColumn("id", substring(col("value"), 6, 7))
```

Now the next wrinkle is that the designations are what they call packed. This means that there is a range of different meanings to the data. There is an explanation in the [MPC documentation](http://www.minorplanetcenter.net/iau/info/PackedDes.html). This code is best put into a function that can be tested separately.

Create a new function. There are three main cases we need to deal with.

* If all the characters in the string are numbers.
* If the third character is a number.
* Any thing else.

Your function should look something like this:

```scala
  def unpackIdFunc(in : String) : String = {
    if (isAllDigits(in.substring(1))) {
      trimZeroFunc(unpackNumbered(in))
    } else if (in(2) >= '0' && in(2) <= '9') {
      numericId(in)
    } else {
      twoCharCode(in)
    }
  }
```

So this function takes in a string and performs out three tests before deciding how to format the result. The first detection of if its an integer is a simple function `isAllDigits()`. I'm going to leave it, along with the `trimZeroFunc()`, as an exercise for the reader.

The Next is the `unpackNumbered` function. This looks like this:

```scala
  def unpackNumbered(in : String) : String = {
    if (in(0) >= '0' && in(0) <= '9') {
      in
    } else {
      val numeric = in.substring(1)
      val expanded = if (in(0) >= 'a' && in(0) <= 'z') {
        in(0) - 'a' + 36
      } else {
        in(0) - 'A' + 10
      }

      expanded.toString + numeric
    }
  }
```

The really simple path in this function is if the first character of the string is a number. In which case we don't have to do anything here. If it is not a number then we need to convert the first character into a number and then append the rest of the input string. The first character uses a range from 0-9 then A-Z followed by a-z to encode numbers 0 to 61. This helps save a bit of space in the files and keeps things in a reasonable order with out having to add an extra leading zero on to the numbers (and change the column lengths) every time too many asteroids are found.

Next up is the `numericId()` function. This one needs to pull some bits from different places and arrange them correctly.

```scala
  def numericId(in : String) : String = {
    val year = unpackNumbered(in.substring(0, 3))
    val number = trimZeroFunc(unpackNumbered(in.substring(4, 6)))
    if (number == "" || number == "00") {
      year + " " + in(3) + in(6)
    } else {
      year + " " + in(3) + in(6) + number
    }
  }
```

The format consists of the year, a space, the forth character in the string, the seventh character in the string and then a packed number between the two. You can see this reuses the `unpackNumbered()` function from above.

Last we have the `twoCharCode()` function. This one is very simple after the last one. Here we just have to unpack a number and then join it up with two characters and some spacers.

```scala
  def twoCharCode(in : String) : String = {
    val number = unpackNumbered(in.substring(3))
    number + " " + in(0) + "-" + in(1)
  }
```

Now we have a function that will unpack the different types of id we might find in the observation file. I recommend using the examples given in the MPC documentation to create some test cases. Put them in `src/test/<your package name>` and you will be able to run them with `./gradlew test` or the correct button in your ide. If you have got this wrong you will get very strange results later.

We need to turn this function `unpackIdFunc` into a user defined function (udf) so that spark knows how to use it. It will need to work out how to send the function to other computers and so on. Thankfully it does a lot of magic behind the scenes so we just need to define it.

```scala
val unpackId = udf(unpackIdFunc _)
```

The underscore here just means that the first parameter to the udf should be passed through to the `unpackIdFunc` function. Now we can use it in our program. Change the line we created earlier to extract the id column to call the unpackId udf.

```scala
  .withColumn("id", unpackId(substring(col("value"), 6, 7)))
```

Nesting functions like this is very powerful and useful. You can often save your self from creating lots of temporary columns by doing this. Though it can sometimes be harder to work out what is going on.

Our original use case was to be able to find the full date of the first observation of an object. So the next thing we need to extract is the date and time of the observation. This starts in column 16 and is 16 characters long. We will also need to create a function to convert the date from a string into an actual date object.

The date format is a little bit weird. The Year, month and day are pretty reasonable. The first four characters are the year, the next two the month and the next two are the day. The rest of the string however is the part of the day divided into 10000 chunks. So we need to work out how many seconds we have if we take the number of seconds in a day and divide it by 10000. Then we can multiply that number by the last part of the string. Take a look at the code below it will hopefully make a little more sense.

```scala
  def dateFunc(in : String): Long = {
    val year = in.substring(0, 4).toInt
    val month = in.substring(5, 7).toInt
    val day = in.substring(8, 10).toInt

    val part = in.substring(11).replaceAll(" ", "0").toInt
    val seconds = Math.round(((24*60*60) * 0.00001) * part)

    LocalDate.of(year, month, day).atStartOfDay()
      .plus(seconds, ChronoUnit.SECONDS)
      .toInstant(ZoneOffset.UTC)
      .getEpochSecond
  }
```

I've used the java 8 date functions as they are a significant improvement over the older API. The bit to watch out for is to make sure you set the time zone to utc. Finally we return it in epoch seconds as spark doesn't know how to handle dates very well. It is just a lot easier to deal with the date a long. Now turn the `dateFunc` into a udf like we did with the `unpackIdFunc`

```scala
 val dateConvert = udf(ObsUtils.dateFunc _)
```

Then add another column to our data frame.

```scala
  .withColumn("ts", dateConvert(substring(col("value"), 16, 16)))
```

We now have the columns we need so we can try and find the minimum time stamp for each id. To do this we need to first group by the id column and then find the minimum of the ts column. We will need to use the `.groupBy()` and `col` functions for the first part. Rather inconsistently the `.min()` function does not need its column name wrapped in a col call. What we end up with should look like this:

```scala
val obs = spark.read.text(observationPath)
      .withColumn("id", unpackId(substring(col("value"), idStart, idLen)))
      .withColumn("ts", dateConvert(substring(col("value"), 16, 16)))
      .groupBy(col("id"))
      .min("ts")
```

There are other aggregation functions available as you would expect. There is a `max`, `avg`, and `sum` to get you started. It is also possible to create your own.

Now you can add a `obs.show(5)` call below it to have a look at the results you have. You should see two columns that look something like below

```text
+-------+-----------+
|     id|    min(ts)|
+-------+-----------+
|1908 OD|-1938820667|
|1914 KA|-1755298440|
|1927 UA|-1331164308|
|1931 RS|-1208486322|
|1933 DC|-1163208817|
+-------+-----------+
only showing top 5 rows
```

The dates are not massively useful like this. Converting them back in to a human readable string requires another user defined function.

```scala
   def formatDateFunc(in : Long) : String = {
    LocalDateTime.ofEpochSecond(in, 0, ZoneOffset.UTC).format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)
  }
```

This uses the built in ISO8601 date formatter. We could use any thing but might as well use the built in standard. Turn it into a UDF as usual.

```scala
  val formatDate = udf(ObsUtils.formatDateFunc _)
```

Last we need to use this to format the ts column. We can use the `.withColumn()` function like before. But this time we are also going to use the `.select()` function to remove the min(ts) column that we don't need any more. This can be very useful if you only need a few columns in a large data set. Also note that we had to call the ts column min(ts) now as it has had its name changed by the min function.

```scala
  .withColumn("date", formatDate(col("min(ts)")))
  .select("id", "date")
```

If you now run `obs.show(5)` you should get something like

```text
+-------+-------------------+
|     id|               date|
+-------+-------------------+
|1908 OD|1908-07-24T22:42:13|
|1914 KA|1914-05-19T01:06:00|
|1927 UA|1927-10-27T00:08:12|
|1931 RS|1931-09-15T21:21:18|
|1933 DC|1933-02-20T22:26:23|
+-------+-------------------+
only showing top 5 rows
```

# Conclusion

In this post we have learnt to read a text file, pull the bits out of the lines that we need, create user defined functions to handle more complex processing of columns, group by columns, perform aggregations, and select columns. This will hopefully leave you in pretty good stead for processing data.  We will join up these two data sets in the next post.

If you found this interesting or you have any questions please let me know on twitter.