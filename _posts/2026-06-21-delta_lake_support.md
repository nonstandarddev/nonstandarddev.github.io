---
title: "Adding Delta Lake to an existing Apache Spark cluster"
date: 2026-06-21
layout: single
categories: [software, apache-spark, delta-lake]
excerpt: >
  An addendum to [my previous post](https://nonstandarddev.github.io/software/apache-spark/2026/06/11/apache_spark_local.html): brief note on how to add Delta Lake functionality to an existing Apache Spark cluster.
---

In [my last post](https://nonstandarddev.github.io/software/apache-spark/2026/06/11/apache_spark_local.html), 
I spoke about how I created a standalone Apache Spark cluster [using Docker Compose](https://github.com/nonstandarddev/spark/tree/main). I'd advise you to read that post before reading this one as there may be context in there
which I gloss over here. Here's the latest [repo](https://github.com/nonstandarddev/spark).

Despite how slick this all is, I do *not* recommend you run enterprise workloads on such a bare architecture. At the very least you would need to provision more resources (i.e. more workers and a larger compute device). In 
practice though, even this extra step wouldn't be enough: how, for instance, are you going to *govern*
all of your assets? How will you cover all possible failure modes? Where are you going to *store* the
input data required to run Spark workloads? How will you integrate this storage mechanism?

This is kind of why Databricks exists. They abstract away all of these complicated (but *real*) 
problems that organisations would need to solve in their absence. Most organisations do *not* have an
IT department that is suitably prioritised (by upper management) to deal with all of these problems. 

Like any vendor worth their salt, Databricks are willing to step in and take responsibility for when things go
wrong. *Unlike* other vendors though, Databricks have given *a lot* back to the community via the development
of open source technology (and they've done this *from the very start*). 

I have a lot of respect for [Matei Zaharia](https://en.wikipedia.org/wiki/Matei_Zaharia) and what 
he has done for the wider community at large. I think other SaaS founders could learn a thing or two 
from Matei. But alas, that is a separate point for another (lengthy) blog post at a future date.

All this said, I still maintain that running a standalone cluster has value,

* Building your own cluster gives you a deeper understanding of how Apache Spark actually works. I 
like learning (a lot), so this reason alone justifies the effort.
* You can experiment with Spark syntax without having to sign up to Databricks or AWS. For some 
people, this may be a (significant) plus.
* Finally, it's fun. 

# Storage Layer: Delta Lake

One (glaring) omission from my last model was a *storage layer*.

Sure, you can use ordinary files on disk and that will suffice for grasping the basics of Apache 
Spark but, as part of my foray into the world of Databricks, I've also started learning more about 
[Delta Lake](https://delta.io/).

To understand Delta Lake, you need to understand the problems it was trying to overcome in the first
place. Essentially, you know how when you write data to a database like SQL Server or Postgres you're 
able to rely on [ACID](https://en.wikipedia.org/wiki/ACID) transactional guarantees? Delta Lake basically implements this for blob storage mechanisms like
S3. There's a lot more to it than that: if you're looking for an indepth examination of the subject 
just read their ['Definitive Guide'](https://delta.io/pdfs/dldg_databricks.pdf) - I'm about a third 
of the way through this and it's been a brilliant read so far.

To add Delta Lake features to my cluster, I've had to make a few changes which I've explained below.

## Note: `deltalake` vs `delta-spark`

If you want to process I/O via Apache Spark, you *must* install [`delta-spark`](https://pypi.org/project/delta-spark/) and then import features with `from delta import ...`. When you use Delta Lake this way, you are letting Spark handle the 
reads / writes to and from your delta tables, in a distributed manner (potentially). 

By contrast, if you *just* want to use Delta Lake independently of Apache Spark, you do *not* need a Spark cluster! Python has a [`deltalake`](https://delta-io.github.io/delta-rs/) package which hooks into Rust (under the hood) and allows you to play around with Delta Tables on whatever device you wish (with whatever computational engine you wish e.g. DuckDB).

As you can imagine, the former approach (`delta-spark`) has more 'setup' involved than the latter. This
is what makes up the bulk of the changes I have presented below.

## Change #1: `Dockerfile`

We basically need to ensure that when we run Spark workloads, our workers have access to the correct
jars (JVM) in order to run distributed I/O.

This can be achieved by adding a line for `spark.jars.packages` to [`./config/spark-defaults.conf`](https://github.com/nonstandarddev/spark/blob/main/config/spark-defaults.conf). Note, however, that I'm adding this line
programmatically - inside my [`Dockerfile`](https://github.com/nonstandarddev/spark/blob/main/Dockerfile) - so as to preserve the versioning scheme,

```dockerfile
...

# ---- 4: Delta Lake: Setup ----

ARG SCALA_VERSION=2.13
ARG DELTA_SPARK_VERSION=4.1.0
ENV DELTA_PACKAGE_VERSION=io.delta:delta-spark_${SPARK_MINOR_VERSION}_${SCALA_VERSION}:${DELTA_SPARK_VERSION}

RUN printf "\nspark.jars.packages ${DELTA_PACKAGE_VERSION}\n" >> ${SPARK_HOME}/conf/spark-defaults.conf

...
```

This extra config option that we add to `spark-defaults.conf` basically tells the cluster to install
the compiled jars for the specified `DELTA_PACKAGE_VERSION`. As we'll see later, you can setup a 
volume to persist this cache beyond the scope of a spark session within a given container.

The other change to my Dockerfile involved adding the `delta-spark` and `deltalake` packages to 
the final installation step (technically, the latter is not necessary),

```dockerfile
...

RUN pip3 install --no-cache-dir \
    pandas \
    pyarrow \
    pyspark==${SPARK_VERSION} \
    delta-spark==${DELTA_SPARK_VERSION} \
    deltalake

...
```

## Change #2: Cluster Configuration (`./config/spark-defaults.conf`)

In addition to the above, I provisioned a [few other necessary configuration options](https://github.com/nonstandarddev/spark/blob/main/config/spark-defaults.conf) inside my `./config/spark-defaults.conf` which, for 
whatever reason, are required in order to 'activate' Delta Lake functionality on the cluster,

```
...
spark.sql.extensions io.delta.sql.DeltaSparkSessionExtension
spark.sql.catalog.spark_catalog org.apache.spark.sql.delta.catalog.DeltaCatalog
...
```

One other option I've added is the following,

```
spark.jars.ivy /opt/spark/ivy-cache
```

This is simply because when Spark installs the aforementioned jars it chooses a default location and
I wanted to create my own custom location under `/opt/spark` (for consistency with `/opt/spark/data` and
`/opt/spark/apps`).

## Change #3: Adding another volume (`compose.yml`)

As mentioned, Spark has the potential to install jars everytime you run a Spark workload; however,
it will use a 'cache' to avoid re-downloading them. The only issue is, this cache only exists in 
the container. If, for whatever reason, the container is stopped or removed, you lose the cache and
it has to be rebuilt once again.

To combat this I created a named *volume* to persist the cached jars permanently inside my [`compose.yml`](https://github.com/nonstandarddev/spark/blob/main/compose.yml) file. Here is a schematic below,

```yaml
  ...

  spark-container:
    ...
    volumes:
          ...
          - spark-ivy:/opt/spark/ivy-cache

volumes:
  ...
  spark-ivy:
```

## Running Workloads

With all of that setup out of the way, you can now process Delta Lake compliant I/O.

I created two example scripts to showcase different approaches for using Delta Lake,

* `delta-spark`: processing I/O via Apache Spark (see: [`delta-lake-example.py`](https://github.com/nonstandarddev/spark/blob/main/spark_apps/delta-lake-example.py))
* `deltalake`: processing I/O purely via Rust (see: [`delta-lake-example-rs.py`](https://github.com/nonstandarddev/spark/blob/main/spark_apps/delta-lake-example-rs.py))
