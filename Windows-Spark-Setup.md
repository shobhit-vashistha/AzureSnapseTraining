# Windows Spark/Scala Setup

## Install MS Open JDK 11 -

- Download installer from https://aka.ms/download-jdk/microsoft-jdk-11.0.23-windows-x64.msi
- Install using the installer
- On custom setup, select the option to add `JAVA_HOME` environment variable
- Make sure the environment variable `JAVA_HOME` is `C:\Program Files\Microsoft\jdk-11.0.23.9-hotspot\`, if it is not, edit it to set to this value
    - Search "env" in the start menu and select "Edit the system environment variables", a new window will open
    - Click "Environment Variables", a new window will open
    - `JAVA_HOME` should be set to `C:\Program Files\Microsoft\jdk-11.0.23.9-hotspot`
- Verify from command line/powershell (if a shell is already open, you would need to close and reopen the shell window)

```
PS C:\Users\shobh> java -version
openjdk version "11.0.23" 2024-04-16 LTS
OpenJDK Runtime Environment Microsoft-9394293 (build 11.0.23+9-LTS)
OpenJDK 64-Bit Server VM Microsoft-9394293 (build 11.0.23+9-LTS, mixed mode, sharing)
```


## Install Maven -
- Download maven zip from https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.zip
- Unzip to `C:\Program Files\` -> now you should have `C:\Program Files\apache-maven-3.9.6` path present
- Add `C:\Program Files\apache-maven-3.9.6\bin` to `PATH` environment variable
    - Search "env" in the start menu and select "Edit the system environment variables", a new window will open
    - Click "Environment Variables", a new window will open
    - From "System Variables" select "Path" and click "Edit", a new window will open
    - Click "New" and add `C:\Program Files\apache-maven-3.9.6\bin`, and click "Ok"
    - Close the other windows
- Verify from command line/powershell (if a shell is already open, you would need to close and reopen the shell window)

```
PS C:\Users\shobh> mvn -version
Apache Maven 3.9.6 (bc0240f3c744dd6b6ec2920b3cd08dcc295161ae)
Maven home: C:\Program Files\apache-maven-3.9.6
Java version: 11.0.23, vendor: Microsoft, runtime: C:\Program Files\Microsoft\jdk-11.0.23.9-hotspot
Default locale: en_IN, platform encoding: Cp1252
OS name: "windows 11", version: "10.0", arch: "amd64", family: "windows"
```

Source: https://maven.apache.org/install.html



## Install Scala 2.13 -
- Download Coursier Installer - https://github.com/coursier/coursier/releases/latest/download/cs-x86_64-pc-win32.zip, unzip and run exe
- If Microsoft Defender warning pops up, click "More Info" > "Run Anyway"
- Wait for the installation to complete, this will install scala 3.4 by default (but it will also install some useful utils)
- Open powershell and run `cs install scala:2.13.14 scalac:2.13.14` to install scala 2.13
- Verify from command line/powershell

```
PS C:\Users\shobh> scala -version
Scala code runner version 2.13.14 -- Copyright 2002-2024, LAMP/EPFL and Lightbend, Inc.
```

Sources:
- https://docs.scala-lang.org/getting-started/index.html#using-the-scala-installer-recommended-way
- https://www.scala-lang.org/download/2.13.14.html



## Install Spark 3.5.1 -

(direct download link https://dlcdn.apache.org/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3-scala2.13.tgz)

- Download Spark at https://spark.apache.org/downloads.html
    - Select Spark release as "3.5.1"
    - Select package type as "Pre-built for Apache Hadoop 3.3 or later (Scala 2.13)"
    - Click the download link
- Extract the file to `C:\Program Files\` (.tgz file, you will need Winrar), you should have `C:\Program Files\spark-3.5.1-bin-hadoop3-scala2.13` path present now
- Add System environment variable `SPARK_HOME` as `C:\Program Files\spark-3.5.1-bin-hadoop3-scala2.13`
- Add `%SPARK_HOME%\bin` to "Path" environment variable
- Install Hadoop 3.3.1 on Windows (needed for spark) -
    - Download https://github.com/robguilarr/spark-winutils-3.3.1/blob/master/hadoop-3.3.1/bin/winutils.exe?raw=true
    - Create folder `C:\winutils\bin` and move the downloaded exe to this folder
    - Add System environment variable `HADOOP_HOME` as `C:\winutils`
    - Add `%HADOOP_HOME%\bin` to "Path" environment variable
- Verify from command line/powershell (if a shell is already open, you would need to close and reopen the shell window)

```
PS C:\Users\shobh> spark-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
____              __
/ __/__  ___ _____/ /__
_\ \/ _ \/ _ `/ __/  '_/
/___/ .__/\_,_/_/ /_/\_\   version 3.5.1
/_/

Using Scala version 2.13.8 (OpenJDK 64-Bit Server VM, Java 11.0.23)
Type in expressions to have them evaluated.
Type :help for more information.
24/05/24 13:03:29 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Spark context Web UI available at http://Optimus:4040
Spark context available as 'sc' (master = local[*], app id = local-1716536012492).
Spark session available as 'spark'.

scala>
```

Source: https://www.knowledgehut.com/blog/big-data/how-to-install-apache-spark-on-windows#apache-spark-installation-system-requirement-for-windows%C2%A0%C2%A0



## Install IntelliJ IDEA (Community or Professional Edition)
- Go to IntelliJ IDEA website, download and install
- Open IntelliJ, and install Scala plugin, go to File > Settings > Plugins, search for "Scala" and install the Scala Plugin



## Create a new Maven/Scala project in Intellij -
- Open Intellij and click File > New > project
- Select "Maven Archetype" from sidebar on the left
- Give a project name, for JDK, select jdk-11 from the dropdown
- In "Archetype" click Add, a window will pop up, set the following values, and click Add
    - groupId: `net.alchim31.maven`
    - artifactId: `scala-archetype-simple`
    - version: `1.7`
- Click "Create", Wait for the project to get created
- Run main > scala > ... > App > main to make sure everything is working so far



## Update project to Scala 2.13 -
- Open "pom.xml" and edit `<scala.version>` tag to reflect `<scala.version>2.13.14</scala.version>`
- update `scala.compat.version` to 2.13
- update `spec2.version` to 4.20.6
- in `<dependencies>` tag update junit version to 4.13.2
- in `<dependencies>` tag update scalatest version to 3.2.18
- click the maven refresh button
- Run main > scala > ... > App > main to make sure everything is working so far

To check compatibility of maven packages search package at - https://mvnrepository.com/repos/central

## Add Spark 3.5.1 dependencies to the project -
- In "pom.xml" add `<spark.version>3.5.1</spark.version>` to `<properties>` tag
- In `<dependencies>` add the following dependencies -

```xml
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-core_${scala.compat.version}</artifactId>
      <version>${spark.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_${scala.compat.version}</artifactId>
      <version>${spark.version}</version>
      <scope>compile</scope>
    </dependency>
```

- Add some spark code to main > scala > ... > App > main


```scala
// to import SparkSession
import org.apache.spark.sql.SparkSession
```

```scala
  val spark = SparkSession.builder.appName("Simple Application").master("local[*]").getOrCreate()
  val values = List(1,2,3,4,5)
  import spark.implicits._
  val df = values.toDF()
  df.show()
  spark.stop()
```


- Run main > scala > ... > App > main to make sure spark code is running

Sample pom.xml file: https://gist.github.com/mahen-github/02ec3c756dd11656dd2fb2f90fce82ce




