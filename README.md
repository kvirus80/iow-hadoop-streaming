iow-hadoop-streaming
====================

Set of hadoop input/output formats for use in combination with hadoop streaming
(http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/HadoopStreaming.html)

Input formats that read Avro or Parquet files and convert them to text or json and 
then feed into streaming MapRed job as input.
Output formats for converting text or json output of streaming MapRed jobs and storing it in Avro or Parquet.
Output format that can write to many files based on record prefix and can be combined with above output formats

Heavily based on code from ASF (http://www.apache.org/) and Twitter (http://twitter.com)

Credits to:

Vladimir Klimontovich <vklimontovich@getintent.com>, Evgeny Terentiev <terentev@gmail.com>

Build with 'mvn package' and put on all your hadoop nodes (and a box which starts the job)
into HADOOP_CLASSPATH (or send with -libjars)

Known limitations: arrays of non-primitive types not supported yet. When using in text* formats, arrays are
still JSON-encoded.
When converting to json, these formats will wrap NaN/Inf into quotes, otherwise json is invalid to many parsers.
When converting from json, at least Avro output formats will try to squeeze "NaN" into float and double fields as real NaN/Inf.
Issue with reading avro was fixed in avro-1.7.7 (https://issues.apache.org/jira/browse/AVRO-1454), but not the issue with writing, as far as I understand. 

Usage examples (assuming iow-hadoop-streaming-1.0.jar is present in HADOOP_CLASSPATH, otherwise include with -libjars):

Reading Avro file:

```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
-D mapreduce.output.fileoutputformat.compress=true -D stream.reduce.output=text \
-inputformat net.iponweb.hadoop.streaming.avro.AvroAsJsonInputFormat \
-input <your input> -output <your output> -mapper <your mapper> -reducer <your reducer>
```


Reading Parquet file:

```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
-D parquet.read.support.class=net.iponweb.hadoop.streaming.parquet.GroupReadSupport \
-D mapreduce.output.fileoutputformat.compress=true -D stream.reduce.output=text \
-inputformat net.iponweb.hadoop.streaming.parquet.ParquetAsJsonInputFormat \
-input <your input> -output <your output> -mapper <your mapper> -reducer <your reducer>
```

Writing Avro file:

```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
-D iow.streaming.output.schema=<your avro schema> -files <your avro schema>
-D mapreduce.output.fileoutputformat.compress=true -D stream.reduce.output=text \
-outputformat net.iponweb.hadoop.streaming.avro.AvroAsJsonOutputFormat \
-input <your input> -output <your output> -mapper <your mapper> -reducer <your reducer>
```

Writing Parquet file:

```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
-D iow.streaming.output.schema=<your parquet schema> -files <your parquet schema>
-D mapreduce.output.fileoutputformat.compress=true -D stream.reduce.output=text \
-D parquet.compression=gzip -D parquet.enable.dictionary=true \
-D parquet.write.support.class=net.iponweb.hadoop.streaming.parquet.GroupWriteSupport \
-outputformat net.iponweb.hadoop.streaming.parquet.ParquetAsJsonOutputFormat \
-input <your input> -output <your output> -mapper <your mapper> -reducer <your reducer>
```

Of course, you can combine any inputformat with any outputformat, just make sure that your
output records correspond to format (text/json) and schema

Using ByKeyOutputFormat is a bit tricky. At first, every file generated by reducer should have
unique name. File name is effectively the first field in your reducer output. (Field separator
is map.output.key.field.separator). Easiest way for this is to add slash and Reducer ID, so you
will have something like this:

```
/job_output_dir/output_type_0/0.avro
/job_output_dir/output_type_0/1.avro
...
```

If your output has single schema, this is enough. If different schemas required, they should be
named before the colon in the first field:

schema0:output_type_0/0

Schemas should be present in job work directory (passed by -files) and iow.streaming.schema.use.prefix
should be set to true (-D iow.streaming.schema.use.prefix=true). Schema name could be 'default',
in that case, schema from net.iponweb.output.streaming.schema is used.
Actual underlying format should be specified in iow.streaming.bykeyoutputformat. Default value is text.
Supported values are:

text, sequence, avrotext, avrojson, parquettext, parquetjson

Example:

```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
-files pq_schema0,pq_schema1 -D mapreduce.output.fileoutputformat.compress=true \
-D stream.reduce.output=text -D parquet.compression=gzip -D parquet.enable.dictionary=true \
-D parquet.write.support.class=net.iponweb.hadoop.streaming.parquet.GroupWriteSupport \
-D iow.streaming.bykeyoutputformat=parquetjson -D iow.streaming.schema.use.prefix=true \
-outputformat net.iponweb.hadoop.streaming.ByKeyOutputFormat \
-input <your input> -output <your output> -mapper <your mapper> -reducer <your reducer>
```


Nikita Makeev, IPONWEB
Please contact me with questions and feedback by e-mail: whale2.box@gmail.com
