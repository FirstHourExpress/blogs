# Introduction
I created a contrived example where the goal is to calculate the average values of soil moisture and soil temp
within a specific time interval, namely the period between planting date and emergence date, for each cropID.

## Simulation

We simulate two dataframes here with this code:

```scala
package similator


import org.apache.spark.sql.{DataFrame, Row, SparkSession}
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions.{col, rand, explode}

import java.time.format.DateTimeFormatter
import java.time.{LocalDate, ZoneOffset}
import java.time.format.DateTimeFormatter
import java.time.{Duration, LocalDate}

import utils.Utils._

object Similator {
  // declaration and definition of function
  def createGSDF(
      spark: SparkSession,
      numRows: Int,
      numUniqueCropIds: Int,
      numUniqueSites: Int,
      startDate: LocalDate,
      endDate: LocalDate
  ): DataFrame = {

    // Create the gsDF data frame
    val gsRows: Seq[Row] = (1 to numRows).map { i =>
      val lat: Double =
        39.876887 + (i % numUniqueSites - numUniqueSites / 2) * 0.01
      val lon: Double =
        -105.03242 + (i % numUniqueSites - numUniqueSites / 2) * 0.01
      val cropId: Int = i % numUniqueCropIds + 1
      // format the dates, this converts from LocalDate to String
      val formatter: DateTimeFormatter =
        DateTimeFormatter.ofPattern("yyyy-MMM-dd")
      val plantingDate: String = startDate
        .plusDays(
          scala.util.Random.nextInt(
            endDate.toEpochDay.toInt - startDate.toEpochDay.toInt + 1
          )
        )
        .format(formatter)
      val emergenceDate: String =
        LocalDate
          .parse(plantingDate)
          .plusDays(scala.util.Random.nextInt(20) + 1)
          .format(formatter)
      Row(lat, lon, cropId, plantingDate, emergenceDate)
    }
    val gsSchema: StructType = StructType(
      Seq(
        StructField("lat", DoubleType, nullable = false),
        StructField("lon", DoubleType, nullable = false),
        StructField("cropID", IntegerType, nullable = false),
        StructField("plantingDate", StringType, nullable = false),
        StructField("emergenceDate", StringType, nullable = false)
      )
    )
    val gsDF: DataFrame =
      spark.createDataFrame(spark.sparkContext.parallelize(gsRows), gsSchema)

    gsDF
  }

  def createSoilDF(
      spark: SparkSession,
      gsDF: DataFrame,
      startDate: LocalDate,
      endDate: LocalDate
  ): DataFrame = {

    // format the dates, this converts from LocalDate to String
    val formatter: DateTimeFormatter =
      DateTimeFormatter.ofPattern("yyyy-MMM-dd")

    // without this line toDF won't work
    import spark.implicits._
    // Generate dates between startDate and endDate
    val minMaxWithRange = getDateRange(startDate, endDate).toDF("range")

    val dates = minMaxWithRange
      .withColumn("date", explode(col("range")))
      .drop("range")

    //  Join the sequence of data with gsDF dataframe
    val soilDF: DataFrame = gsDF
      .select(col("lat").as("lat"), col("lon").as("lon"))
      .distinct()
      .crossJoin(dates)
      .withColumn("soilMoisture", rand())
      .withColumn("soilTemp", rand(18) * 6 + 18)


    // Return the soilDF data frame
    soilDF
  }

}

```

### GS Dataframe

| lat      | lon       | cropID | plantingDate | emergenceDate |
|----------|-----------|--------|--------------|---------------|
| 39.87689 | -105.0324 | 1      | 2022-Jan-04  | 2022-Jan-12   |
| 38.0000  | -100.0314 | 2      | 2022-Jan-20  | 2022-Feb-02   |
| 40.84300 | -102.1374 | 3      | 2022-Jan-12  | 2022-Jan-29   |
| ...      | ...       | ...    | ...          | ...           |

The `gs` dataframe has columns for latitude, longitude, crop ID, planting date, and emergence date. It is generated using the `createGSDF()` function in the `Simulator` object, which takes in a number of rows, number of unique crop IDs, number of unique sites, start date, and end date. It then generates random values for latitude, longitude, crop ID, planting date, and emergence date, and returns a dataframe with those columns.


### Soil Dataframe

| lat      | lon       | date       | soilMoisture | soilTemp |
|----------|-----------|------------|--------------|----------|
| 39.87689 | -105.0324 | 2022-01-01 | 0.318        | 20.063   |
| 39.87689 | -105.0324 | 2022-01-02 | 0.762        | 18.944   |
| 39.87689 | -105.0324 | 2022-01-03 | 0.436        | 23.832   |
| ...      | ...       | ...        | ...          | ...      |

The `soil` dataframe has columns for latitude, longitude, date, soil moisture, and soil temperature. It is generated using the `createSoilDF()` function in the `Simulator` object, which takes in a `gsDF` (generated using `createGSDF()`) and generates a cross join with a sequence of dates between the start and end dates passed in. The `soilMoisture` and `soilTemp` columns are then generated using the `rand()` function from the `functions` module of `spark.sql`. The resulting dataframe has a row for every combination of latitude, longitude, and date.
