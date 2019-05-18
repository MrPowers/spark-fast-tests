# spark-fast-tests

A fast Apache Spark testing helper library with beautifully formatted error messages!  Works with scalatest and uTest.

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/ab42211c18984740bee7f87c631a8f42)](https://www.codacy.com/app/MrPowers/spark-fast-tests?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=MrPowers/spark-fast-tests&amp;utm_campaign=Badge_Grade) [![Join the chat at https://gitter.im/spark-fast-tests/community](https://badges.gitter.im/spark-fast-tests/community.svg)](https://gitter.im/spark-fast-tests/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

The `assertSmallDatasetEquality` method can be used to compare two Datasets (or two DataFrames).

```scala
val sourceDF = Seq(
  (1),
  (5)
).toDF("number")

val expectedDF = Seq(
  (1, "word"),
  (5, "word")
).toDF("number", "word")

assertSmallDataFrameEquality(sourceDF, expectedDF)
// throws a DatasetSchemaMismatch exception
```

The `assertSmallDatasetEquality` method can also be used to compare Datasets.

```scala
val sourceDS = Seq(
  Person("juan", 5),
  Person("bob", 1),
  Person("li", 49),
  Person("alice", 5)
).toDS

val expectedDS = Seq(
  Person("juan", 5),
  Person("frank", 10),
  Person("li", 49),
  Person("lucy", 5)
).toDS
```

![assert_small_dataset_equality_error_message](https://github.com/MrPowers/spark-fast-tests/blob/master/images/assertSmallDatasetEquality_error_message.png)

The colors in the error message make it easy to identify the rows that aren't equal.

The `DatasetComparer` has `assertSmallDatasetEquality` and `assertLargeDatasetEquality` methods to compare either Datasets or DataFrames.

If you only need to compare DataFrames, you can use `DataFrameComparer` with the associated `assertSmallDataFrameEquality` and `assertLargeDataFrameEquality` methods.  Under the hood, `DataFrameComparer` uses the `assertSmallDatasetEquality` and `assertLargeDatasetEquality`.

*Note : comparing Datasets can be tricky since some column names might be given by Spark when applying transformations. 
Use the `ignoreColumnNames` boolean to skip name verification.*

## Setup

**Option 1: Maven**

Fetch the JAR file from Maven.

```scala
resolvers += "Spark Packages Repo" at "http://dl.bintray.com/spark-packages/maven"

// Scala 2.11
libraryDependencies += "MrPowers" % "spark-fast-tests" % "0.17.1-s_2.11"

// Scala 2.12, Spark 2.4+
libraryDependencies += "MrPowers" % "spark-fast-tests" % "0.17.1-s_2.12"
```

[Here's a link to all the JAR files in Maven](https://mvnrepository.com/artifact/MrPowers/spark-fast-tests?repo=spark-packages).

**Option 2: JitPack**

Update your `build.sbt` file as follows.

```scala
resolvers += "jitpack" at "https://jitpack.io"
libraryDependencies += "com.github.mrpowers" % "spark-fast-tests" % "v0.16.0" % "test"
```

**Spark version compatibility by spark-fast-tests version**

|       | 0.16.0             | 0.17.0             |
|-------|--------------------|--------------------|
| 2.0.0 | :white_check_mark: | :white_check_mark: |
| 2.1.0 | :white_check_mark: | :white_check_mark: |
| 2.2.2 | :white_check_mark: | :white_check_mark: |
| 2.3.0 | :white_check_mark: | :white_check_mark: |
| 2.3.1 | :white_check_mark: | :white_check_mark: |
| 2.4.0 | :white_check_mark: | :white_check_mark: |

Scala 2.12 support is only for Spark 2.4+.

## Why is this library fast?

The `assertSmallDataFrameEquality` method runs 31% faster than the `assertLargeDatasetEquality` method as described in [this blog post](https://medium.com/@mrpowers/how-to-cut-the-run-time-of-a-spark-sbt-test-suite-by-40-52d71219773f).

The `assertSmallDataFrameEquality` method uses the Dataset `collect()` method, which is a lot faster than the RDD `zipWithIndex()` method that's used by the other Spark testing libraries (and the `assertLargeDatasetEquality()` method).

spark-fast-tests also provides a `assertColumnEquality()` method that's even faster and easier to use!

## Usage

The spark-fast-tests project doesn't provide a SparkSession object in your test suite, so you'll need to make one yourself.

```scala
import org.apache.spark.sql.SparkSession

trait SparkSessionTestWrapper {

  lazy val spark: SparkSession = {
    SparkSession
      .builder()
      .master("local")
      .appName("spark session")
      .config("spark.sql.shuffle.partitions", "1")
      .getOrCreate()
  }

}
```

It's typically best set the number of shuffle partitions to one in your test suite.  This configuration can make your tests run up to 70% faster.  You can remove this configuration option or adjust it if you're working with big DataFrames in your test suite.

Make sure to only use the `SparkSessionTestWrapper` trait in your test suite.  You don't want to use test specific configuration (like one shuffle partition) when running production code.

The `DatasetComparer` trait defines the `assertSmallDatasetEquality` method.  Extend your spec file with the `SparkSessionTestWrapper` trait to create DataFrames and the `DatasetComparer` trait to make DataFrame comparisons.

```scala
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row
import org.apache.spark.sql.functions._
import com.github.mrpowers.spark.fast.tests.DatasetComparer

class DatasetSpec extends FunSpec with SparkSessionTestWrapper with DatasetComparer {

  import spark.implicits._

  it("aliases a DataFrame") {

    val sourceDF = Seq(
      ("jose"),
      ("li"),
      ("luisa")
    ).toDF("name")

    val actualDF = sourceDF.select(col("name").alias("student"))

    val expectedDF = Seq(
      ("jose"),
      ("li"),
      ("luisa")
    ).toDF("student")

    assertSmallDatasetEquality(actualDF, expectedDF)

  }
}
```

To compare large DataFrames that are partitioned across different nodes in a cluster, use the `assertLargeDatasetEquality` method.

```scala
assertLargeDatasetEquality(actualDF, expectedDF)
```

`assertSmallDatasetEquality` is faster for test suites that run on your local machine.  `assertLargeDatasetEquality` should only be used for DataFrames that are split across nodes in a cluster.

### Column Equality

The `assertColumnEquality` method can be used to assess the equality of two columns in a DataFrame.

Suppose you have the following DataFrame with two columns that are not equal.

```
+-------+-------------+
|   name|expected_name|
+-------+-------------+
|   phil|         phil|
| rashid|       rashid|
|matthew|        mateo|
|   sami|         sami|
|     li|         feng|
|   null|         null|
+-------+-------------+
```

The following code will throw a `ColumnMismatch` error message:

```scala
assertColumnEquality(df, "name", "expected_name")
```

![assert_column_equality_error_message](https://github.com/MrPowers/spark-fast-tests/blob/master/images/assertColumnEquality_error_message.png)

Mix in the `ColumnComparer` trait to your test class to access the `assertColumnEquality` method:

```scala
import com.github.mrpowers.spark.fast.tests.ColumnComparer

object MySpecialClassTest
    extends TestSuite
    with ColumnComparer
    with SparkSessionTestWrapper {

    // your tests
}
```

### Unordered DataFrame equality comparisons

Suppose you have the following `actualDF`:

```
+------+
|number|
+------+
|     1|
|     5|
+------+
```

And suppose you have the following `expectedDF`:

```
+------+
|number|
+------+
|     5|
|     1|
+------+
```

The DataFrames have the same columns and rows, but the order is different.

`assertSmallDataFrameEquality(sourceDF, expectedDF)` will throw a `DatasetContentMismatch` error.

We can set the `orderedComparison` boolean flag to `false` and spark-fast-tests will sort the DataFrames before performing the comparison.

`assertSmallDataFrameEquality(sourceDF, expectedDF, orderedComparison = false)` will not throw an error.

### Equality comparisons ignoring the nullable flag

You might also want to make equality comparisons that ignore the nullable flags for the DataFrame columns.

Here is how to use the `ignoreNullable` flag to compare DataFrames without considering the nullable property of each column.

```scala
val sourceDF = spark.createDF(
  List(
    (1),
    (5)
  ), List(
    ("number", IntegerType, false)
  )
)

val expectedDF = spark.createDF(
  List(
    (1),
    (5)
  ), List(
    ("number", IntegerType, true)
  )
)

assertSmallDatasetEquality(sourceDF, expectedDF, ignoreNullable = true)
```

### Approximate DataFrame Equality

The `assertApproximateDataFrameEquality` function is useful for DataFrames that contain `DoubleType` columns.  The precision threshold must be set when using the `assertApproximateDataFrameEquality` function.

```scala
val sourceDF = spark.createDF(
  List(
    (1.2),
    (5.1),
    (null)
  ), List(
    ("number", DoubleType, true)
  )
)

val expectedDF = spark.createDF(
  List(
    (1.2),
    (5.1),
    (null)
  ), List(
    ("number", DoubleType, true)
  )
)

assertApproximateDataFrameEquality(sourceDF, expectedDF, 0.01)
```

## Testing Tips

* Use column functions instead of UDFs as described in [this blog post](https://medium.com/@mrpowers/spark-user-defined-functions-udfs-6c849e39443b)
* Try to organize your code as [custom transformations](https://medium.com/@mrpowers/chaining-custom-dataframe-transformations-in-spark-a39e315f903c) so it's easy to test the logic elegantly
* Don't write tests that read from files or write files.  Dependency injection is a great way to avoid file I/O in you test suite.

## uTest settings to display color output

Create a `CustomFramework` class with overrides that turn off the default uTest color settings.

```scala
package com.github.mrpowers.spark.fast.tests

class CustomFramework extends utest.runner.Framework {
  override def formatWrapWidth: Int = 300
  // turn off the default exception message color, so spark-fast-tests
  // can send messages with custom colors
  override def exceptionMsgColor = toggledColor(utest.ufansi.Attrs.Empty)
  override def exceptionPrefixColor = toggledColor(utest.ufansi.Attrs.Empty)
  override def exceptionMethodColor = toggledColor(utest.ufansi.Attrs.Empty)
  override def exceptionPunctuationColor = toggledColor(utest.ufansi.Attrs.Empty)
  override def exceptionLineNumberColor = toggledColor(utest.ufansi.Attrs.Empty)
}
```

Update the `build.sbt` file to use the `CustomFramework` class:

```scala
testFrameworks += new TestFramework("com.github.mrpowers.spark.fast.tests.CustomFramework")
```

## Alternatives

The [spark-testing-base](https://github.com/holdenk/spark-testing-base) project has more features (e.g. streaming support) and is compiled to support a variety of Scala and Spark versions.

You might want to use spark-fast-tests instead of spark-testing-base in these cases:

* You want to use uTest or a testing framework other than scalatest
* You want to run tests in parallel (you need to set `parallelExecution in Test := false` with spark-testing-base)
* You don't want to include hive as a project dependency
* You don't want to restart the SparkSession after each test file executes so the suite runs faster

## Publishing

Build the JAR / POM files with `sbt +spDist` as described in [this GitHub issue](https://github.com/databricks/sbt-spark-package/issues/18#issuecomment-184107369).

Manually upload the zip files to [Spark Packages](https://spark-packages.org/).

Make a GitHub release so the code is available via JitPack.

## Additional Goals

* Use memory efficiently so Spark test runs don't crash
* Provide readable error messages
* Easy to use in conjunction with other test suites
* Give the user control of the SparkSession

## Contributing

Open an issue or send a pull request to contribute.  Anyone that makes good contributions to the project will be promoted to project maintainer status.

