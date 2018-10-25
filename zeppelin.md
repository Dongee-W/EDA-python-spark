# Spark interpreter documentation
https://zeppelin.apache.org/docs/0.8.0/interpreter/spark.html

# Settings
Copy `conf/zeppelin-env.sh.template` to `conf/zeppelin-env.sh`
```
export SPARK_HOME=/Users/summerlight/App/spark-2.3.0-bin-hadoop2.7 
```

# Zeppelin build-in plotting
* Histogram
  ```
  val histogram = rgm_renamed.select("amount").as[Double].rdd.histogram(Math.sqrt(rgm_renamed.count).toInt)
  val plotting_data = histogram._1.zip(histogram._2)
    .map(a => a._1 + "\t" + a._2)
    .mkString("\n")

  println(s"""%table
    |amount\tfrequency
    |$plotting_data
  """.stripMargin)
  ```

# EDA in zeppelin
Dependency
```
%spark.dep
z.load("com.typesafe.play:play-json_2.11:2.6.10")
```
Imports
```
import play.api.libs.json._
import play.api.libs.functional.syntax._
import org.apache.spark.sql.Dataset
import org.apache.spark.sql.Column
```
Basic Template
```
val randomIndex = Math.abs(scala.util.Random.nextInt)
print(s"""%html
<head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
<body><div id="plotCat_${randomIndex}"></div></body>
<script>
    var data = [
    {
        x: ${category(df, column)._1},
        y: ${category(df, column)._2},
        type: 'bar',
        marker: {
            color: '#90353B'
        }
    }
    ];

    Plotly.newPlot('plotCat_${randomIndex}', data);
</script>
""")  
```
Basic information
```
val tips = spark.read.
  option("delimiter", ",").
  option("header", "true").
  option("inferSchema", "true").
  csv("/Users/summerlight/Desktop/tips.csv")

tips.schema.foreach(a => println(a))

z.show(tips)

z.show(tips.summary())

// Count missing value
z.show(tips.select(tips.columns.map(c => sum(col(c).isNull.cast("int")).alias(c)): _*))
```
Plotting setup
```
val the_economist = List("#90353B", "#1A476F", "#008BBC", "#9C8847")
val boxplot = List("#3D9970", "#FF4136", "#FF851B", "rgb(107,174,214)")
def getColor(index: Int, flavor: Int = 0) = {
    if (flavor == 1) {
        val boxplot = List("#3D9970", "#FF4136", "#FF851B", "rgb(207, 114, 255)")
        val size = boxplot.size
        val adjustIndex = index %  size
        boxplot(adjustIndex)
    } else {
        val the_economist = List("#90353B", "#1A476F", "#008BBC", "#9C8847")
        val size = the_economist.size
        val adjustIndex = index %  size
        the_economist(adjustIndex)
    }

}
```
## Univariate plots
### Categorical
```
def plotCategory[S](df: Dataset[S], column: String) = {
    def category[S](df: Dataset[S], column: String) = {
        val statistics = df.select(col(column)).groupBy(col(column)).
            count().as[(String, Long)].collect
        val x = statistics.map(r => r._1)
        val y = statistics.map(r => r._2)
        (Json.toJson(x), Json.toJson(y))
    }
    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCat_${randomIndex}"></div></body>
    <script>
	  var data = [
        {
          x: ${category(df, column)._1},
          y: ${category(df, column)._2},
          type: 'bar',
          marker: {
              color: '#90353B'
          }
        }
      ];
      var layout = {
          title: "$column Count",
          xaxis: {
              title: "$column"
          },
          yaxis: {
              title: "count"
          }
      }

      Plotly.newPlot('plotCat_${randomIndex}', data, layout);
    </script>
  """)  
}
plotCategory(tips, "sex")
```
### Numerical
```
def plotNumeric[S](df: Dataset[S], column: String) = {
    def numeric[S](df: Dataset[S], column: String, bins: Int = 400) = {
        val statistics = df.select(column).as[Double].rdd.histogram(bins)
        val x = Json.toJson(statistics._1.dropRight(1))
        val y = Json.toJson(statistics._2)
        (Json.toJson(x), Json.toJson(y))
    }
    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotNu_${randomIndex}"></div></body>
    <script>
        var trace = {
            x: ${numeric(df, column)._1},
            y: ${numeric(df, column)._2},
            autobinx: false, 
            type: 'histogram',
            histfunc: "sum",
            marker: {
                color: "#1A476F", 
                line: {
                  color:  "rgba(255, 255, 255, 0.1)", 
                  width: 1
                }
              },
          };
        var data = [trace];
        
        var layout = {
            title: "$column Distribution",
            xaxis: {
              title: "$column"
            },
            yaxis: {
              title: "count"
          }
        }
        Plotly.newPlot('plotNu_${randomIndex}', data, layout);
    </script>
    """)
}
plotNumeric(tips, "total_bill")
```
## Bivariate plots
### Categorical & categorical
Simple - bucket + sub-bucket
```
def catVsCat[S](df: Dataset[S], bucket: String, subBucket: String) = {
    val statistics = df.select(bucket, subBucket).
        groupBy(bucket, subBucket).count().
        as[(String, String, Long)].collect
    val bucketElement =  statistics.map(x => x._1).distinct
    val n = bucketElement.size
    val subBucketElement =  statistics.map(x => x._2).distinct
    val lookupTable = statistics.map(x => (x._1, x._2) -> x._3).toMap.withDefaultValue(0L)
    val summation = for {
        b <- bucketElement
    } yield subBucketElement.map(x => (b, x, lookupTable((b, x))))
    
    def createJson(summation: Array[Array[(String, String, Long)]]) = {
        val traces = summation.zipWithIndex.map{
            x => 
            
                val xaxisFormat = if (x._2 != 0) ("x" + (x._2 + 1).toString) else "x"
                Json.obj(
                    "x" -> Json.toJson(x._1.map(y => y._2)),
                    "y" -> Json.toJson(x._1.map(y => y._3)),
                    "xaxis" -> xaxisFormat,
                    "yaxis" -> "y",
                    "type" -> "bar",
                    "name" -> x._1.head._1,
                    "marker" -> Json.obj(
                        "color" -> getColor(x._2)
                        )
                )
        }
        Json.toJson(traces)
    }
    def layout(n: Int, subBucket: String) = {
        def suffix(i: Int) = if (i != 1) i.toString else ""
        val array = Json.toJson(for (i <- 1 to n) yield {
            "x" + suffix(i) + "y"
        })
        val layout = Json.obj(
            "grid" -> Json.obj("rows" -> 1, "columns" -> 4),
            "subplots" -> array,
            "title" -> s"Group by $bucket",
            "yaxis" -> Json.obj(
                "title" -> "count"
                )
        )
        val axisTitle = for (i <- 1 to n) yield {
            Json.obj(
                ("xaxis" + suffix(i)) -> Json.obj(
                    "title" -> subBucket
                )
            )
        }
        axisTitle.foldLeft(layout)((x, y) => x ++ y)
    }
    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotcatCat_${randomIndex}"></div></body>
    <script>
        var data = ${createJson(summation)};

        var layout = ${layout(n, subBucket)};

        Plotly.newPlot('plotcatCat_${randomIndex}', data, layout);
    </script>
    """)
}
catVsCat(tips, "day", "time")
```
Simple - count matrix
```
def catVsCatMatrix[S](df: Dataset[S], verticalAxis: String, horizontalAxis: String) = {
    val statistics = df.select(verticalAxis, horizontalAxis).
        groupBy(verticalAxis, horizontalAxis).count().
        as[(String, String, Long)].collect
    val vertical =  statistics.map(x => x._1).distinct
    val horizontal =  statistics.map(x => x._2).distinct
    val lookupTable = statistics.map(x => (x._1, x._2) -> x._3).toMap.withDefaultValue(0L)
    val z = Json.toJson(for {
        b <- vertical
    } yield Json.toJson(horizontal.map(x => lookupTable((b, x)))))

    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCatMat_${randomIndex}"></div></body>
    <script>
	var data = [
        {
            z: $z,
            x: ${Json.toJson(horizontal)},
            y: ${Json.toJson(vertical)},
            type: 'heatmap',
            colorscale: 'Portland',
        }
    ];
    
    var layout = {
        title: "Count matrix",
        xaxis: {
            title: "$horizontalAxis"
        },
        yaxis: {
            title: "$verticalAxis"
        }
    };

    Plotly.newPlot("plotCatMat_${randomIndex}", data, layout);
    </script>
    """)  
}
catVsCatMatrix(tips, "day", "time")
```
Compact
```
def catVsCatAll[S](df: Dataset[S], column: String*) = {
    def suffix(i: Int) = if (i != 1) i.toString else ""
    val columns = column.toList.sorted
    val dimension = columns.size
    val axisSharing = {
        val xaxis = for {
            i <- 1 to dimension;
            j <- 1 to dimension
        } yield if (j != 1) "x" + j else "x"
        val yaxis = for {
            i <- 1 to dimension;
            j <- 1 to dimension
        } yield if (i != 1) "y" + (1 + (i-1) * dimension) else "y"
        xaxis.zip(yaxis)
    }
    def pairwiseTrace[S](verticalAxis: String, horizontalAxis: String, index: Int) = {
        val statistics = df.select(verticalAxis, horizontalAxis).
        groupBy(verticalAxis, horizontalAxis).count().
        as[(String, String, Long)].collect
        val vertical =  statistics.map(x => x._1).distinct.sorted
        val horizontal =  statistics.map(x => x._2).distinct.sorted
        val lookupTable = statistics.map(x => (x._1, x._2) -> x._3).toMap.withDefaultValue(0L)
        val z = Json.toJson(for {
            b <- vertical
        } yield Json.toJson(horizontal.map(x => lookupTable((b, x)))))
        Json.obj(
            "z" -> z,
            "x" -> Json.toJson(horizontal),
            "y" -> Json.toJson(vertical),
            "xaxis" -> ("x" + suffix(index + 1)),
            "yaxis" -> ("y" + suffix(index + 1)),
            "type" -> "heatmap",
             "colorscale" -> "Portland"
            )
    }
    val layout = Json.obj(
        "grid" -> Json.obj("rows" -> dimension, 
            "columns" -> dimension,
            "pattern" -> "independent",
            "roworder" -> "bottom to top"
            ),
        "title" -> "All-pair category plot"
    )
    val axisTitle = for (i <- 1 to dimension) yield {
        Json.obj(
            ("xaxis" + suffix(i)) -> Json.obj(
                "title" -> columns(i - 1)
            ),
            ("yaxis" + suffix((1 + (i - 1) * dimension))) -> Json.obj(
                "title" -> columns(i - 1)
            )
        )
    }
    
    
    val data = for {
        i <- columns.zipWithIndex;
        j <- columns.zipWithIndex
    } yield pairwiseTrace(i._1, j._1, i._2 * dimension + j._2)

    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCat_${randomIndex}" style="height:800px;"></div></body>
    <script>


    var layout = ${axisTitle.foldLeft(layout)((x, y) => x ++ y)};

    var data = ${Json.toJson(data)};

    Plotly.newPlot('plotCat_${randomIndex}', data, layout);
    </script>
    """)  
}
catVsCatAll(tips, "sex", "time", "day", "smoker")

```
### Categorical & numerical
Simple - boxplot
```
def catVsNum[S](df: Dataset[S], catColumn: String, numColumn: String) = {
    val category = df.select(catColumn).distinct.as[String].collect.zipWithIndex
    val traces = Json.toJson(for (cat <- category) yield {
        val dfCat = df.select(catColumn, numColumn).filter(col(catColumn) === cat._1)
        val quantile = dfCat.stat.approxQuantile(numColumn, Array(0.25, 0.5, 0.75), 0.0001)
        val minMax = dfCat.agg(min(numColumn), max(numColumn)).as[(Double, Double)].collect
        val data = List(minMax(0)._1, quantile(0), quantile(0), quantile(1), quantile(2), quantile(2), minMax(0)._2)
        Json.obj(
            "y" -> Json.toJson(data),
            "type" -> "box",
            "name" -> cat._1,
            "marker" -> Json.obj(
                        "color" -> getColor(cat._2, 1)
                        )
            )
    })
    val layout = Json.obj(
            "xaxis" -> Json.obj(
                "title" -> catColumn
                ),
            "yaxis" -> Json.obj(
                "title" -> numColumn
                )
        )

    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCatNum_${randomIndex}"></div></body>
    <script>

    var data = ${traces};
    
    var layout = ${layout};

    Plotly.newPlot("plotCatNum_${randomIndex}", data, layout);
    </script>
    """)  
}
catVsNum(tips, "day", "total_bill")
```
Compact
```
def catVsNumAll[S](df: Dataset[S], catColumn: List[String], numColumn: List[String]) = {
    val catDimension = catColumn.size
    val numDimension = numColumn.size
    def suffix(i: Int) = if (i != 1) i.toString else ""
    val randomIndex = Math.abs(scala.util.Random.nextInt)
    def pairwiseTrace[S](catColumn: String, numColumn: String, index: Int) = {
        val category = df.select(catColumn).distinct.as[String].collect.zipWithIndex
        val traces = for (cat <- category) yield {
            val dfCat = df.select(catColumn, numColumn).filter(col(catColumn) === cat._1)
            val quantile = dfCat.stat.approxQuantile(numColumn, Array(0.25, 0.5, 0.75), 0.0001)
            val minMax = dfCat.agg(min(numColumn), max(numColumn)).as[(Double, Double)].collect
            val data = List(minMax(0)._1, quantile(0), quantile(0), quantile(1), quantile(2), quantile(2), minMax(0)._2)
            Json.obj(
                "y" -> Json.toJson(data),
                "xaxis" -> ("x" + suffix(index + 1)),
                "yaxis" -> ("y" + suffix(index + 1)),
                "type" -> "box",
                "name" -> cat._1,
                "marker" -> Json.obj(
                    "color" -> getColor(cat._2, 1)
                    )
                )
        }
        traces
    }
    val catColumnIndex = catColumn.zipWithIndex
    val numColumnIndex = numColumn.zipWithIndex
    val traces = Json.toJson(for {
        num <- numColumnIndex;
        cat <- catColumnIndex;
        trace <- pairwiseTrace(cat._1, num._1, num._2 * numDimension + cat._2 )
    } yield trace)
    
    val layout = Json.obj(
        "grid" -> Json.obj("rows" -> numDimension, 
            "columns" -> catDimension,
            "pattern" -> "independent",
            "roworder" -> "bottom to top"
            ),
        "title" -> "Category numeric plot"
    )
    val axisTitleCat = for (i <- 1 to catDimension) yield {
        Json.obj(
            ("xaxis" + suffix(i)) -> Json.obj(
                "title" -> catColumn(i - 1)
            )
        )
    }
    val axisTitleNum = for (i <- 1 to numDimension) yield {
        Json.obj(
            ("yaxis" + suffix((1 + (i - 1) * catDimension))) -> Json.obj(
                "title" -> numColumn(i - 1)
            )
        )
    }
    val finalLayout = axisTitleNum.foldLeft(axisTitleCat.foldLeft(layout)((x, y) => x ++ y))((x, y) => x ++ y)
    
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCatNumAll_${randomIndex}"  style="height:800px;"></div></body>
    <script>
        var data = ${traces};
        var layout = ${finalLayout};

        Plotly.newPlot('plotCatNumAll_${randomIndex}', data, layout);
    </script>
    """)
}
catVsNumAll(tips, List("day", "time", "sex"), List("total_bill", "tip", "size"))
```
### Numerical & numerical
Simple - 2D histogram
```
def numVsNum[S](df: Dataset[S], xColumn: String, yColumn: String, resolution: Int = 20, zLogScale: Boolean = true) = {
    def toBucket(input: Double, min: Double, max: Double, numBucket: Int) = {
        require (max > min, "max has to be greater than min")
        val size = (max - min) / numBucket
        if (input < min) min - size
        else if (input > max) max + size
        else ((input - min) / size).floor * size + min
    }

    def buckets(min: Double, max: Double, numBucket: Int) = {
        require (max > min, "max has to be greater than min")
        (min to max by (max - min) / numBucket).toArray.dropRight(1)
    }
    
    val stats = df.stat.approxQuantile(Array(xColumn, yColumn), Array(0.02, 0.98), 0.001)
    val binning = df.select(xColumn, yColumn).as[(Double, Double)].
        map(a => (toBucket(a._1, stats(0)(0), stats(0)(1), resolution), toBucket(a._2, stats(1)(0), stats(1)(1), resolution)))
    val histogramData = binning.groupBy("_1", "_2").count().as[(Double, Double, Long)]
    val data = if (zLogScale) histogramData.collect().map(x => (x._1, x._2) -> Math.log(x._3)).toMap.withDefaultValue(0.0)
        else histogramData.collect().map(x => (x._1, x._2) -> x._3.toDouble).toMap.withDefaultValue(0.0)
        
    val xaxis = buckets(stats(0)(0), stats(0)(1), resolution)
    val yaxis = buckets(stats(1)(0), stats(1)(1), resolution)
    val histogram2d = Json.toJson(for (i <- yaxis) yield 
        for (j <- xaxis) yield Json.toJson(data((j, i))))

    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotNumNum_${randomIndex}"></div></body>
    <script>
    var z = []

    var data = [
    {
        z: $histogram2d,
        x: ${Json.toJson(xaxis)},
        y: ${Json.toJson(yaxis)},
        colorscale: 'YIGnBu',
        type: 'heatmap'
    }
    ];
    
    var layout = {
        title: "2d Histogram (zlog=${zLogScale})",
        xaxis: {
            title: "$xColumn"
        },
        yaxis: {
            title: "$yColumn"
        }
    };

    Plotly.newPlot('plotNumNum_${randomIndex}', data, layout);
    </script>
    """)
}
//numVsNum(tips, "tip", "total_bill", 30, false)
numVsNum(pubg, "walkDistance", "winPlacePerc", 50)
```
Compact - 2D histogram
```
def numVsNumAll[S](df: Dataset[S], column: List[String], resolution: Int = 20, zLogScale: Boolean = true) = {
    def suffix(i: Int) = if (i != 1) i.toString else ""
    val columns = column.toList.sorted
    val dimension = columns.size
    val axisSharing = {
        val xaxis = for {
            i <- 1 to dimension;
            j <- 1 to dimension
        } yield if (j != 1) "x" + j else "x"
        val yaxis = for {
            i <- 1 to dimension;
            j <- 1 to dimension
        } yield if (i != 1) "y" + (1 + (i-1) * dimension) else "y"
        xaxis.zip(yaxis)
    }
    def pairwiseTrace[S](xColumn: String, yColumn: String, index: Int) = {
        def toBucket(input: Double, min: Double, max: Double, numBucket: Int) = {
            require (max > min, "max has to be greater than min")
            val size = (max - min) / numBucket
            if (input < min) min - size
            else if (input > max) max + size
            else ((input - min) / size).floor * size + min
        }

        def buckets(min: Double, max: Double, numBucket: Int) = {
            require (max > min, "max has to be greater than min")
            (min to max by (max - min) / numBucket).toArray.dropRight(1)
        }
        
        val stats = df.stat.approxQuantile(Array(xColumn, yColumn), Array(0.02, 0.98), 0.001)
        val binning = df.select(xColumn, yColumn).as[(Double, Double)].
            map(a => (toBucket(a._1, stats(0)(0), stats(0)(1), resolution), toBucket(a._2, stats(1)(0), stats(1)(1), resolution)))
        val histogramData = binning.groupBy("_1", "_2").count().as[(Double, Double, Long)]
        val data = if (zLogScale) histogramData.collect().map(x => (x._1, x._2) -> Math.log(x._3)).toMap.withDefaultValue(0.0)
            else histogramData.collect().map(x => (x._1, x._2) -> x._3.toDouble).toMap.withDefaultValue(0.0)
            
        val xaxis = buckets(stats(0)(0), stats(0)(1), resolution)
        val yaxis = buckets(stats(1)(0), stats(1)(1), resolution)
        val histogram2d = Json.toJson(for (i <- yaxis) yield 
            for (j <- xaxis) yield Json.toJson(data((j, i))))

        Json.obj(
            "z" -> histogram2d,
            "x" -> Json.toJson(xaxis),
            "y" -> Json.toJson(yaxis),
            "xaxis" -> ("x" + suffix(index + 1)),
            "yaxis" -> ("y" + suffix(index + 1)),
            "colorscale" -> "YIGnBu",
             "type" -> "heatmap"
            )

    }
    val layout = Json.obj(
        "grid" -> Json.obj("rows" -> dimension, 
            "columns" -> dimension,
            "pattern" -> "independent",
            "roworder" -> "bottom to top"
            ),
        "title" -> "All-pair 2D histogram"
    )
    val axisTitle = for (i <- 1 to dimension) yield {
        Json.obj(
            ("xaxis" + suffix(i)) -> Json.obj(
                "title" -> columns(i - 1)
            ),
            ("yaxis" + suffix((1 + (i - 1) * dimension))) -> Json.obj(
                "title" -> columns(i - 1)
            )
        )
    }
    
    
    val data = for {
        i <- columns.zipWithIndex;
        j <- columns.zipWithIndex
    } yield pairwiseTrace(i._1, j._1, i._2 * dimension + j._2)

    val randomIndex = Math.abs(scala.util.Random.nextInt)
    print(s"""%html
    <head><script src="https://cdn.plot.ly/plotly-latest.min.js"></script></head>
    <body><div id="plotCat_${randomIndex}" style="height:800px;"></div></body>
    <script>


    var layout = ${axisTitle.foldLeft(layout)((x, y) => x ++ y)};

    var data = ${Json.toJson(data)};

    Plotly.newPlot('plotCat_${randomIndex}', data, layout);
    </script>
    """)  
}
numVsNumAll(pubg, List("walkDistance", "damageDealt", "winPlacePerc"), 50)

```
# Python interpreter settings
```
zeppelin.python  /Users/summerlight/pythonenv/jupyterlab/bin/python
```

# Spark interpreter loading dependency
Common dependency
```
%spark.dep

z.reset() 
// add maven repository
z.addRepo("Q Central").url("http://central.maven.org/maven2/")

z.load("com.github.haifengl:smile-scala_2.11:1.5.2")
z.load("joda-time:joda-time:2.10")
z.load("com.typesafe.play:play-json_2.11:2.6.10")
```



        def toBucket(input: Double, min: Double, max: Double, numBucket: Int) = {
            require (max > min, "max has to be greater than min")
            val size = (max - min) / numBucket
            if (input < min) min - size
            else if (input > max) max + size
            else ((input - min) / size).floor * size + min
        }

        def buckets(min: Double, max: Double, numBucket: Int) = {
            require (max > min, "max has to be greater than min")
            (min to max by (max - min) / numBucket).toArray.dropRight(1)
        }
        
        val stats = df.stat.approxQuantile(Array(xColumn, yColumn), Array(0.02, 0.98), 0.001)
        val binning = df.select(xColumn, yColumn).as[(Double, Double)].
            map(a => (toBucket(a._1, stats(0)(0), stats(0)(1), resolution), toBucket(a._2, stats(1)(0), stats(1)(1), resolution)))
        val histogramData = binning.groupBy("_1", "_2").count().as[(Double, Double, Long)]
        val data = if (zLogScale) histogramData.collect().map(x => (x._1, x._2) -> Math.log(x._3)).toMap.withDefaultValue(0.0)
            else histogramData.collect().map(x => (x._1, x._2) -> x._3.toDouble).toMap.withDefaultValue(0.0)
            
        val xaxis = buckets(stats(0)(0), stats(0)(1), resolution)
        val yaxis = buckets(stats(1)(0), stats(1)(1), resolution)
        val histogram2d = Json.toJson(for (i <- yaxis) yield 
            for (j <- xaxis) yield Json.toJson(data((j, i))))

        val randomIndex = Math.abs(scala.util.Random.nextInt)