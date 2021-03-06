# ADAM Analytics

Demonstrating how to do analytics on ADAM variants data.

## Install tools

First install ADAM

```bash
git clone https://github.com/bigdatagenomics/adam.git
cd adam
export MAVEN_OPTS='-Xmx512m -XX:MaxPermSize=128m'
mvn clean package -DskipTests
```

Then install Parquet tools (not needed if you are using CDH).

```bash
curl -s -L -O http://search.maven.org/remotecontent?filepath=com/twitter/parquet-tools/1.6.0rc3/parquet-tools-1.6.0rc3-bin.tar.gz
tar zxf parquet-tools-*-bin.tar.gz
```

And [Kite tools](http://kitesdk.org/docs/1.0.0/Install-Kite.html) (not needed if you are using CDH).

## Set up the environment

```bash
export ADAM_HOME=~/adam
export SPARK_HOME=/opt/cloudera/parcels/CDH-5.3.0-1.cdh5.3.0.p0.30/lib/spark 
export PARQUET_HOME=/opt/cloudera/parcels/CDH-5.3.0-1.cdh5.3.0.p0.30/lib/parquet
export PATH=$PATH:$ADAM_HOME/bin:$PARQUET_HOME/bin
```

## Copy some genomics data to the cluster

We'll start with just the chr22 VCF from 1000 genomes:

```bash
curl -s -L ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20110521/ALL.chr22.phase1_release_v3.20101123.snps_indels_svs.genotypes.vcf.gz \
 | gunzip \
 | hadoop fs -put - genomics/1kg/vcf/ALL.chr22.phase1_release_v3.20101123.snps_indels_svs.genotypes.vcf
```

Convert it to Parquet using ADAM. Note that the number of executors should reflect the size of your cluster.

```bash
adam-submit --master yarn-cluster --driver-memory 4G --num-executors 24 --executor-cores 2 --executor-memory 4G \
  vcf2adam  genomics/1kg/vcf/ALL.chr22.phase1_release_v3.20101123.snps_indels_svs.genotypes.vcf genomics/1kg/parquet/chr22  
```

Flatten with ADAM, so that we can use Impala to query later on.

```bash
adam-submit --master yarn-cluster --driver-memory 4G --num-executors 24 --executor-cores 2 --executor-memory 4G \
  flatten genomics/1kg/parquet/chr22 genomics/1kg/parquet/chr22_flat
```

## (Optional) Use pre-converted Eggo datasets

Instead of running the conversion ourselves from VCF to Parquet, we can use pre-converted Parquet datasets from the [Eggo project](https://github.com/bigdatagenomics/eggo), which is quicker.

Set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in your shell, then run:

```bash
hadoop distcp \
  -D fs.s3n.awsAccessKeyId=$AWS_ACCESS_KEY_ID \
  -D fs.s3n.awsSecretAccessKey=$AWS_SECRET_ACCESS_KEY \
  -update \
  s3n://bdg-eggo/1kg/genotypes 1kg/genotypes
```

## Register the data in the Hive metastore

This is most easily done using the Kite tools.

First retrieve the flattened schema from one of the files:

```bash
parquet-schema hdfs://bottou01-10g.pa.cloudera.com/user/tom/genomics/1kg/parquet/chr22_flat/part-r-00001.gz.parquet | grep 'extra:' meta.txt
```

Copy the value of `avro.schema` into a new file called genotype_flat.avsc. And then store it in HDFS (we need to do this so that the schema is stored as a reference to a file in HDFS, rather than its literal value, since the latter is too large for the Hive metastore):

```bash
hadoop fs -mkdir schemas
hadoop fs -put genotype_flat.avsc schemas/genotype_flat.avsc
```

Create the Kite dataset in Hive:

```bash
kite-dataset delete dataset:hive:genotypes
kite-dataset create dataset:hive:genotypes --schema hdfs://bottou01-10g.pa.cloudera.com/user/tom/schemas/genotype_flat.avsc \
  --format parquet
```

Move the data into the Kite dataset:

```bash
hadoop fs -mv genomics/1kg/parquet/chr22_flat/*.parquet /user/hive/warehouse/genotypes
```

### TODO: Alternative way

The latest (trunk) Kite code makes this step simpler, since it's now possible to create
a dataset from existing Parquet files in one step.
See https://issues.cloudera.org/browse/CDK-902

```
export HADOOP_CONF_DIR=/etc/hadoop/conf
export HIVE_CONF_DIR=/etc/hive/conf
# build https://github.com/tomwhite/kite.git, branch adam-fixes
```

## Use Impala to query the dataset

Start an Impala shell by typing `impala-shell`. Then run the following so that Impala picks up the new data, and then computes statistics on it.

```sql
invalidate metadata;
compute stats genotypes;
```

Try some queries:
```sql
select count(*) from genotypes;
select * from genotypes limit 1;
select count(*) from genotypes where variant__start > 50000000 and variant__end < 51000000;
select min(variant__start),  max(variant__start) from genotypes;
```

## Use Spark to do a range lookup

Start an ADAM shell (which starts a Spark shell with the ADAM libraries already registered):

```bash
ADAM_OPTS='--conf spark.hadoop.parquet.task.side.metadata=false' adam-shell --master yarn-client --executor-memory 4G
```

In the shell, try the following:

```
import org.bdgenomics.adam.rdd.ADAMContext._
import parquet.filter2.dsl.Dsl._
// following line does not work due to https://issues.apache.org/jira/browse/PARQUET-173
// val pred = (LongColumn("variant.start") > 50000000L && LongColumn("variant.end") < 50000060L)
val pred = LongColumn("variant.start") > 50000000L
val samples = sc.loadParquetGenotypes("genomics/1kg/parquet/chr22", predicate = Some(pred))
samples.take(2).foreach(println(_))
```




