---
title: "Building a standalone Apache Spark cluster"
date: 2026-06-11
layout: single
categories: [software, apache-spark]
excerpt: >
  An overview of what Apache Spark *actually is* followed by a practical
  walkthrough of building a local standalone cluster with Docker Compose
---

I recently embarked on the journey of trying to 'learn Databricks', primarily because my employer has
been kind enough to support my journey towards the [Machine Learning Associate](https://www.databricks.com/learn/certification/machine-learning-associate) 
qualifcation over the next year or so.

The trouble is, I've been having trouble defining what people *mean* by 'Databricks'. What *is* Databricks? 
Seriously: ask the person next to you, then ask someone else later on - you're bound to get different 
answers (such is the life of those who work in the nebulous field of 'data').

The truth is, Databricks is an absolutely *massive* data platform that encapsulates a plethora of software frameworks.
Each of these frameworks focus on a different set of problems. It's not like learning 'Python' or learning 'R' - think more along the lines of 'learning an 
*ecosystem* of various packages' that comprise what we now know recognise as 'Databricks'.

To be transparent, whilst I do have experience of *using* Databricks to execute Python code,
I don't think I ever ever really understood what exactly was going on under the hood. I distinctly remember firing up 
Databricks for the first time and being completely bemused as to why it took so long to execute 
such a seemingly simple (small) Python script. 

As it turns out, there's a big reason for this and it's to do with a core architecture on which
the Databricks platform is built: Apache Spark.

# Apache Spark: 101

To give you a brief overview of what Apache Spark actually *is*: essentially it's a distribution of 
software (no more than [a tarball](https://dlcdn.apache.org/spark/), really) that is geared towards 
the *distributed*, parallel processing of 'big' data. If you've ever used Anaconda before (god rest
your soul), you should be somewhat familiar with the concept of a 'software distribution'.

Note well: we're talking *big* data here (i.e. data which cannot reasonably fit on one node). As 
a rough benchmark: on most modern machines, if you're trying to process datasets which are larger than 
~ 100GB in pure volume, then you may want to start considering frameworks like Spark to process 
them more efficiently. That is, unless you have access to [a HPC](https://en.wikipedia.org/wiki/High-performance_computing), 
in which case, go wild: you clearly aren't like the rest of us plebs!

Anything smaller than 'big', and the benefits of Spark start to diminish. As we'll see shortly, Spark doesn't 
come for free and there *is* an overhead associated with provisioning services on a cluster (hence
my earlier 'bemused' puzzled look at the computer screen when it took Databricks 5 seconds to run
a one-liner in Python).

Medium sized data which *does* fit on one machine can be treated with modern solutions like DuckDB and/or 
Polars which is written in Rust. For smaller data still... sure, I *guess you can use Pandas* but the unintuitive syntax
is enough to drive anyone insane - does anyone actually remember how to aggregate a bloomin' dataframe in Pandas *without looking at the docs*? And do *not* forget to `.reset_index()` - I'm watching you!

Anyway, back to Spark. The architecture of Spark is surprisingly simple:

![Figure: architectural diagram depicting Apache Spark]({{ "/assets/images/architecture.png" | relative_url }})

You can ignore the reference to Docker for now - I'll explain that in a minute. Spark calls the collection of resources on the RHS (i.e. the driver, cluster manager / master and the worker nodes) a *cluster*. Some notes on this,

* You write some 'syntax' into an executable script and/or notebook. Spark supplies a surprising variety of APIs: you can use any of Scala, Java, Python (`pyspark`) or R (`sparkr`) 
* The script you've written *is not executed end-to-end right away*. Spark is 'smart' about things and utilises lazy evaluation under the hood: it won't actually execute anything until you 'tell' it to do something (e.g. `.collect()` the results). Generally, Spark - and specifically the *driver* node - pauses, sees what you're trying to do and then comes up with an effective 'plan of attack'. For example,
    * If your script involves 'selecting' certain columns well, the driver can come up with a plan which says 'Aha! You only care about these three columns. Therefore, I'm gonna completely ignore all of the other columns when I read the data.'
    * Equally, if you apply filters later on in the script which could easily be performed at the start, it might say 'OK let me try to apply your filter directly to the paritioned data rather than read it all into memory first (and so on...)'
* Whilst the driver node can 'plan' and act as the effective 'brain' of the cluster, it doesn't execute anything. That's up to the *workers* on the RHS of the diagram. Jobs are allocated to workers via a separate *master* node which is responsible for 'managing' computational resources effectively

If you're with me so far then, hopefully, you're starting to feel like you understand Apache Spark a little more concretely. 

# Running Apache Spark (Locally)

I felt that, as part of my Databricks training, it would be instructive for me to learn Apache Spark and to do that I needed some kind of playground.

Of course, you can just use Amazon's EMR or the Databricks Community Edition, both of which would come at minimal cost and overhead but... that wouldn't allow me to satisfy the insatiable desire of 'building something from scratch'. 

So, I decided to look into whether I could provision a cluster locally (on my desktop) via Docker which could then, in theory, be scaled to a more powerful machine in the future, should I so wish to run Spark with a little more firepower. In practice, though, you should just use a platform like Databricks if you're getting to that stage. But this was fun and I wanted to share the fruits of my discovery!

## Containerised Cluster

### Repository Link

__TLDR;__ here is [the repo](https://github.com/nonstandarddev/spark/tree/main) - `git clone` followed by `make run` and you should be golden. 

In what follows, I will provide some useful context for the _contents_ of this repo so that you might, you know, learn something?

### The Dockerfile

The core idea here is to run each 'node' inside a different container.

For example, we'll have one 'client / driver' node, one 'master' node and one or more 'worker' nodes. 
I'm also going to add an (optional) 'history' node which is essentially an additional service that 
allows us to examine a logged history of Spark jobs.

Before we can provision this cluster (using Docker Compose), we *first* need a base Docker image that contains the Apache Spark distribution. This is non-negotiable and must exist in every single node in the cluster. 

As such, here is the base image template for all nodes,

```docker
# ---- 1: User-Defined Parameters ----

ARG PYTHON_VERSION=3.11

# ---- 2: Build Tools: Setup ----

FROM python:${PYTHON_VERSION}-bullseye AS spark-image

ARG JDK_VERSION=17
ARG JDK_SRC=openjdk-${JDK_VERSION}-jdk

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ${JDK_SRC} \
        rsync \
        build-essential \
        software-properties-common \
        ssh && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ---- 3: Apache Spark: Setup ----

ARG SPARK_VERSION=4.1.2
ARG SPARK_SRC=https://dlcdn.apache.org/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop3.tgz

ENV SPARK_HOME=/opt/spark
ENV SPARK_MASTER_HOST=spark-master
ENV SPARK_MASTER_PORT=7077
ENV SPARK_MASTER_URL="spark://${SPARK_MASTER_HOST}:${SPARK_MASTER_PORT}"
ENV PATH=${SPARK_HOME}/sbin:${SPARK_HOME}/bin:${PATH}
ENV PYSPARK_PYTHON=python3

RUN mkdir -p ${SPARK_HOME}

WORKDIR ${SPARK_HOME}

RUN curl -fSL "${SPARK_SRC}" -o spark-dist.tgz && \
    tar xvzf spark-dist.tgz --directory "${SPARK_HOME}" --strip-components 1 && \
    rm -f spark-dist.tgz

# NB: by default, jobs log at the `INFO` level; I find this too noisy
COPY config/log4j2.properties ${SPARK_HOME}/conf/log4j2.properties

# NB: by default, *event* logs are disabled; I include overrides here to reverse this
COPY config/spark-defaults.conf ${SPARK_HOME}/conf/spark-defaults.conf

# NB: we need to ensure bash scripts like `start-master.sh` are executable at runtime
RUN chmod u+x ./sbin/* && \
    chmod u+x ./bin/*

# ---- 4: Python: Setup ----

# NB: we do not need to install `pyspark` as this will be invoked via 
#     the `spark-submit` binaries stored within ${SPARK_HOME}; however, it is helpful 
#     to do so for intellisense purposes!

RUN pip3 install \
    ipython \
    pandas \
    pyspark==${SPARK_VERSION} \
    pyarrow

COPY provision.sh .

RUN chmod u+x provision.sh

ENTRYPOINT ["./provision.sh"]
```

So, we basically,

* Start with a compatible Python version (e.g. `3.11`)
* Install a suitable JDK (since Spark is built on Scala / Java and this is required)
* Download the latest Apache Spark distribution
* Copy in some useful config variables (via `./config`) - see below
* Make some 'shell' scripts executable (e.g. `/opt/spark/sbin/start-master.sh`) - see below
* Install some basic development packages for Python - this could easily be supplanted by usage of `requirements.txt` or `uv.lock`. I may update the repository in time to do this (so if the repo looks a bit different when you come to read this blog post, you'll know I've done that already!)
* Set an entrypoint for any running containers: `./provision.sh` - see below

### Configuration Files

As discussed above, I found it useful to set certain configurations for 'ease of use',

* `./config/log4j2.properties` ([link](https://github.com/nonstandarddev/spark/blob/main/config/log4j2.properties)) ensures that Spark does not, by default, log process output at the `INFO` level. This 
is a _personal preference_ but I found the logs to be excessive at the `INFO` level so I changed all of
them to only spit out logs at the `ERROR` level, by default
* `./config/spark-defaults.conf` ([link](https://github.com/nonstandarddev/spark/blob/main/config/spark-defaults.conf)) ensures that *events* (e.g. running a Spark script) can be logged 
for the history server to consume

### The Entrypoint

The `./provision.sh` entrypoint is quite simple,

```sh
#!/bin/bash

SPARK_MECHANISM=$1

if [[ "$SPARK_MECHANISM" == "master" ]]; then
  start-master.sh -h $SPARK_MASTER_HOST -p $SPARK_MASTER_PORT
elif [[ "$SPARK_MECHANISM" == "worker" ]]; then
  start-worker.sh $SPARK_MASTER_URL
elif [[ "$SPARK_MECHANISM" == "history" ]]; then
  start-history-server.sh
else
   echo "Provided mechanism '$SPARK_MECHANISM' not recognised (must be one of: 'master', 'worker' or 'history')"
fi
```

The bash scripts contained within (e.g. `start-master.sh`) are supplied by Apache Spark as part of 
the distribution.

### Service Shell Scripts

Each of the aforementioned bash scripts, when invoked, essentially 'start' the requested service.

For example, 

```sh
./sbin/start-master.sh  # NB: relative to wherever you've unzipped the distribution tarball
```

This will, by default, start a 'master' server running on port `7077`, with a web UI exposed on port `8080`.

You can optionally disable the default behaviour of services to run as daemons but setting 
`SPARK_NO_DAEMONIZE=true`. As we'll come to see this is important so that we can keep all of the 
containers 'running' (otherwise the containers will stop).

### Scaling with Docker Compose

With all of that admin out of the way, we're almost at the finish line. The final step is to tie 
all of these concepts together into a cluster which can be achieved fairly trivially with Docker 
Compose as below,

```yaml
services:

  spark-master:
    container_name: spark-master
    build: .
    image: spark-image
    entrypoint: ["./provision.sh", "master"]
    healthcheck:
      test: [ "CMD", "curl", "-f", "http://localhost:8080"]
      interval: 5s
      timeout: 5s
      retries: 3
    env_file:
      - .env.spark
    volumes:
      - ./spark_data:/opt/spark/data
      - ./spark_apps:/opt/spark/apps
      - spark-logs:/opt/spark/spark-events
    ports:
      - "8080:8080" # Frontend
      - "7077:7077" # Backend

  spark-worker:
    container_name: spark-worker
    build: .
    image: spark-image
    entrypoint: ["./provision.sh", "worker"]
    depends_on:
      - spark-master
    env_file:
      - .env.spark
    volumes:
      - ./spark_data:/opt/spark/data
      - ./spark_apps:/opt/spark/apps
      - spark-logs:/opt/spark/spark-events
    ports:
      - "8081:8081"

  spark-history:
    container_name: spark-history
    build: .
    image: spark-image
    entrypoint: ["./provision.sh", "history"]
    depends_on:
      - spark-master
    env_file:
      - .env.spark
    volumes:
      - spark-logs:/opt/spark/spark-events
    ports:
      - "18080:18080"
  
  spark-developer:
    container_name: spark-developer
    build: .
    image: spark-image
    entrypoint: ["/bin/bash"]
    stdin_open: true
    tty: true
    depends_on:
      - spark-master
      - spark-worker
    volumes:
      - ./spark_apps:/workspace/spark_apps:cached
      - ./spark_data:/workspace/spark_data:cached
      - ./spark_data:/opt/spark/data
      - ./spark_apps:/opt/spark/apps
      - spark-logs:/opt/spark/spark-events

volumes:
  spark-logs:
```

I do ensure that every container is equipped with a `.env.spark` file. This is a tiny file 
with the following key-value pairs,

```
SPARK_NO_DAEMONIZE=true
SPARK_PUBLIC_DNS=localhost
```

Technically speaking, you could probably set these environment variables in the image itself. The 
first environment variable, discussed earlier, ensures that services do not 'stop' once they are 
administered. The second environment variable ensures that the `stdout` logs associated with jobs 
can be accessed on the workers (though I've found this to be unnecessary).

## Testing the Cluster

### Setup

To get this up and running, you need to ensure that you have Docker installed on your system.

At that point, you can then run `docker compose up` or, if you've cloned the repo and you have 
GNU Make installed on your local machine, you can run,

```sh
make run
```

### Navigation

At this stage, simply go to your browser and navigate to `localhost:8080` and you _should_ be brought
to the web UI for your _master_ node as per below,

![Figure: web UI for master node in Spark]({{ "/assets/images/spark-master.png" | relative_url }})

Similarly, you can access your _history_node by navigating to `localhost:18080` as per below,

![Figure: web UI for history node in Spark]({{ "/assets/images/spark-history.png" | relative_url }})

### Executing Spark Jobs

You may have noticed in my compose file the existence of an _additional_ 'Spark Developer' container.

Strictly speaking, this is not necessary in order to run Spark workloads - I just like doing so for 
reproducibility purposes and ensuring that _my_ environment will always work for me (and you!).

To run Spark Jobs, you just need access to the CLI submission tool `spark-submit`, which is contained 
within the Spark distribution (and my developer container, by extension). You _can_ also use a REST 
API service to invoke Spark Jobs but I couldn't be bothered with that for now.

Inside any environment - whether that be your local desktop or the developer container above - that
has the `spark-submit` CLI you can submit Spark workloads like so,

```sh
spark-submit \
    --master spark://spark-master:7077 \
    --deploy-mode client \
    "/opt/spark/apps/enter_your_script_name.py" 
```

Note `--deploy-mode client`: this means the _driver_ is running on your client, which is a sensible
default setting. 

You can change this setup to work with [Spark Connect](https://spark.apache.org/docs/latest/spark-connect-overview.html) which effectively shifts the driver from the client to the cluster so your Python client becomes 'thinner', only dealing in 
RPC terms - note, however, that this approach restricts syntax to DataFrame-like operations 
rather than traditional RDD style, hence my reluctance to introduce that limitation early on. If you 
_are_ exclusively writing DataFrame-like operations, the benefits of using Spark Connect 
may be justified.

Because submitting Spark Jobs is quite a routine action (and it can be cumbersome to write the above
over and over again), I created a [shell script](https://github.com/nonstandarddev/spark/blob/main/spark_apps/submit.sh) as per below,

```sh
#!/bin/bash
set -euo pipefail

if [ "$#" -lt 1 ]; then
    echo "Usage: $0 <script.py> [script args...]"
    exit 1
fi

script="$1"
shift

spark-submit \
    --master spark://spark-master:7077 \
    --deploy-mode client \
    "/opt/spark/apps/$script" \
    "$@"
```
