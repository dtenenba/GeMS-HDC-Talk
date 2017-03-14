class: center, middle
name: title

## GeMS Pipeline Optimization For Geraghty Lab

## Dan Tenenbaum, Scientific Computing

<h3 id="display_time"></h3>

???

These are presenter notes (that's what the ??? indicates).

See the talk at (production URL)
https://dtenenba.github.io/GeMS-HDC-Talk/
To build locally, do:

    gulp

To deploy to production url above (probably have to commit/push first?):

    gulp deploy-pages    

If that's not enough, see further instructions at
https://github.com/brenopolanski/remark-boilerplate
https://github.com/gnab/remark

---
name: overview

## Overview

The GeMS pipeline is a project that originated in the
[Geraghty Lab](http://www.fredhutch.org/en/labs/profiles/geraghty-daniel.html).

The Geraghty Lab reached out to
[SciComp](https://teams.fhcrc.org/sites/citwiki/SciComp/Pages/Home.aspx?TreeField=Wiki_x0020_Page_x0020_Categories) for help modernizing the codebase.

SciComp took on this work (a longer-term project than we usually help
out with) in order to learn more about modern cloud stacks in the context
of a real project.

The result of that work is what we are presenting today.



---

## What is the GeMS Pipeline?



* GeMS is  <b>Ge</b>netics <b>M</b>anagement <b>S</b>oftware
* Geraghty Lab manufactures special reagent kits
  for use with the MiSeq sequencer & makes these
  available to external collaborators.
* In order to make sense of sequencing results,
  a special analysis pipeline is needed.
* Input (fastq files from illumina sequencer)
* Output (a report that can be viewed in GeMS-UI desktop app)
* HLA Typing - immune receptor genes
   human leukocyte antigen
* Results are for donor matching (mostly bone marrow)
* Used by transplant centers
* Increasing accuracy of past results
* Extensible to infectious disease, autoimmune disorders





???

they sell reagent kits
for people to use in their own labs with miseq
they generate a dna library for the sequencer
to understand the results, we need to run it through


pictures - they may have some pictures


---

## Architecture of the Pipeline (old version), part 1

<!-- TODO: need Brandon/Wyatt to confirm my understanding of this -->

![old pipeline part 1](img/old_pipeline_1.jpg)


---

## Architecture of the Pipeline (old version), part 2

### Inside a single worker

![old pipeline part 2](img/old_pipeline_2.jpg)

---

## Final Output

A compressed XML file:

<img src="img/xml.png" width="400" height="200" />

...viewed in the GeMS UI desktop app:

<img src="img/gems-ui.png" width="400" height="200" />

---

class: small
exclude: true

## Old Pipeline Components

&nbsp;

| Component | Description |
| --- | --- |
| [Illumina MiSeq Sequencer](https://www.illumina.com/systems/sequencing-platforms/miseq.html) | Produces initial data (in fastq.gz format) |
| [Amazon S3](https://aws.amazon.com/s3/) | File storage at all stages of the pipline |
| [Amazon SimpleDB](https://aws.amazon.com/simpledb/) | Deprecated key-value store, used for queueing and storing job metadata. SimpleDB was not designed to be a queueing system. |
| [BaseSpace](https://basespace.illumina.com/home/index) | Illumina's app gateway and file storage space, one of two ways to kick off the pipeline. |
| [ScisCloud Web Application](https://sciscloud.herokuapp.com/) | A second way to kick off the pipeline. |
| Queueing Server | An always-on [EC2](https://aws.amazon.com/ec2/) instance listening for new jobs, either via a Python web server (for jobs from BaseSpace) or SSH connections (for jobs from ScisCloud) |
| Worker Pool | A group of stopped EC2 instances, some of which are started when jobs come in. |
| Python Code (private repository) | The glue that holds everything together and orchestrates the steps in the pipeline. |
| Java Code  (private repository) | Code that performs the actual HLA typing  |
| [cross_match](http://www.bioinformatics.org/wiki/Phred_Phrap_Consed_Cross_match) (C program) | Third-party tool that compares DNA sequence sets. |



---
exclude: true

## Updating the Pipeline

The [Geraghty lab](www.fredhutch.org/en/labs/profiles/geraghty-daniel.html)
approached [SciComp](https://teams.fhcrc.org/sites/citwiki/SciComp/Pages/Home.aspx?TreeField=Wiki_x0020_Page_x0020_Categories)
for help with refactoring the pipeline. Together we identified areas that could be improved:

* Queueing server is always on, even though requests to start
  jobs come in infrequently. Geraghty lab is paying by the hour
  for a machine that is mostly idle.
* SimpleDB is used to store job metadata, and essentially as a
  queueing system. However, SimpleDB is now deprecated, and
  was never designed to be a queueing system.
* There is no automated or formalized process for creating the
  Amazon Machine Image (AMI) upon which the workers are based.
* Having a finite pool of workers is problematic and makes jobs
  run slower than they need to, since the number of workers that
  can run at any one time is limited. There are still charges
  for stopped instances.
* There is no canonical process for building Java code and
  distributing build products.
* It is difficult to locally develop code that is designed to
  run inside a worker.
* Codebase is aging and many of the people who worked on it
  have moved on.
* It is difficult to learn about the status of running jobs.    

???

split this into 2 slides
maybe with themes: technology vs process

make a table with columns: problem / solution
(maybe 2 pages of that)
use underline or bold to emphasize key points

---

class: small

## Updating the Pipeline, part 1: technology

Problem | Solution
--- | ---
Queueing server is always on, even though requests to start jobs come in infrequently. Geraghty lab is paying by the hour for a machine that is mostly idle. | AWS Lambda is "serverless" compute, springing into existence only when we need it.
SimpleDB is used to store job metadata, and essentially as a queueing system, to queue jobs onto a pool of EC2 instances that are stopped when idle. However, SimpleDB is now deprecated, and  was never designed to be a queueing system. | Get rid of pool of stopped workers, start workers from scratch as needed. No more need for a queueing system. Job metadata is stored in EC2 instance tags. Each worker processes 3 (or fewer) samples serially (as this can be done in less than an hour), but any number of workers can run in parallel.
It is difficult to learn about the status of running jobs. | [REST API](#REST-API) makes it easy.
If something goes wrong, it can be hard to find out which worker failed and what exactly happened. | Using [Watchtower](https://pypi.python.org/pypi/watchtower), ordinary Python log messages go into [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) where they can be viewed in aggregate.
---

class: small


## Updating the Pipeline, part 2: Process

Problem | Solution
--- | ---
There is no automated or formalized process for creating the Amazon Machine Image (AMI) upon which the workers are based. | Process is now captured in a shell script that runs when each worker starts, _bootstrapping_ it.
There is no canonical process for building Java code and distributing build products. | Continuous Integration (CI): Use [CircleCI](https://circleci.com/) to automatically build Java code on each commit and deploy to S3 along with a commit tag.
It is difficult to locally develop code that is designed to run inside a worker. | [Docker](https://www.docker.com/) container and [Unit Tests](https://docs.python.org/2/library/unittest.html): Container is created using the same script that _bootstraps_ workers. AWS Services and REST APIs are mocked out in unit tests, allowing local development/testing of code that runs on workers.
Codebase is aging and many of the people who worked on it have moved on. | The process of updating the project touched a lot of the code, and we tried to modify all touched files to be free of [linter](#best-practices) errors and warnings.
Credentials and other secrets are hardcoded in source files checked into version control. | Not advised even in private GitHub repositories, solved using [CredStash](#best-practices)

---

exclude: true

## New Pipeline Components

| Component | Description |
| --- | --- |
| [AWS Lambda](https://aws.amazon.com/lambda/) Trigger | "Serverless" Compute. Only used when a triggering event happens. Replaces the queueing server.  Also provides RESTful interface to current job information. |
| EC2 Instance Tags | Used to store job metadata. Replaces SimpleDB. |
| EC2 [User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) Script | Provides a reproducible and consistent way to provision the workers. |
| Horizontal Scaling | Launch as many workers as needed. Replaces the worker pool. |
| Continuous Integration (CI) | Use [CircleCI](https://circleci.com/) to automatically build Java code on each commit and deploy to S3 along with a commit tag.
| [Docker](https://www.docker.com/) container and [Unit Tests](https://docs.python.org/2/library/unittest.html) | Container is created using the same script that provisions workers. AWS Services and REST APIs are mocked out in unit tests, allowing local development/testing of code that runs on workers. |
| Code Refactor | Rewrote substantial parts of the codebase with an eye towards best practices. |

???

Jobs are kicked off either with a [Django](https://www.djangoproject.com/)
web app or from [BaseSpace](https://basespace.illumina.com/home/index).

The process of "kicking off" a job involves `ssh`'ing to a 'queueing'
server (an [AWS EC2](https://aws.amazon.com/ec2/)
instance that is always running) and running a [Python](https://www.python.org/)
script (`Enqueue.py`). This script sends meta-information about the job
to [AWS SimpleDB](https://aws.amazon.com/simpledb/). Then some workers are
started. The 'worker pool' is a group of 18 (??) EC2 instances which are
normally in the `stopped` state but are changed to `running` when a job starts.
at which time they look in SimpleDB for information about what they should do.


---

## How AWS Lambda is used in the pipeline (part 1)

![usage of AWS Lambda](img/lambda_diagram.jpg)

---


## How AWS Lambda is used in the pipeline (part 2)

* [Lambda](https://aws.amazon.com/lambda/) lets you define "Lambda functions" that can be written in
  Python, Java, or Node.js. (Ours is in Python.)
* These functions can be triggered by [numerous events](https://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html).
* Using [Amazon API Gateway](https://aws.amazon.com/api-gateway/), you can assign a URL to a Lambda function,
  and trigger the function by calling that URL.
* Using a tool called [Zappa](https://github.com/Miserlou/Zappa),
  you can write an ordinary [Flask](http://flask.pocoo.org/) application
  and deploy it as a Lambda function. During development you can test, run, and debug the app on your local machine.
* In our case, the Flask application has  RESTful endpoints (to be consumed by workers
  and interested clients such as webapps).
* Amazon has released its [Python Serverless Microframework for AWS](https://aws.amazon.com/blogs/developer/preview-the-python-serverless-microframework-for-aws/) (PSMA) which exposes a framework similar to,
  but different from, Flask. We prefer to use Flask directly,
  so we use Zappa. This also avoids vendor lock-in.
  Here's [a more detailed comparison](https://gist.github.com/dtenenba/f695a6ddd29fbe900a440773083563a7), and [a longer one](https://blog.zappa.io/posts/comparison-zappa-verus-chalice) from one of the Zappa authors.

???

make a diagram of how lambda is used
list advantages of lambda
link to https://gist.github.com/dtenenba/f695a6ddd29fbe900a440773083563a7 to show
comparison
does flask work in other cloud's lambda equivalent? (google)
flask is used in app engine (but we don't know if it works in google
cloud functions)

---

## Advantages of AWS Lambda (or any 'serverless' framework)

* No paying for server that is always on but rarely used.
* No maintenance (updating packages, patching, etc.)
* Focus exclusively on business logic.
* Numerous [event sources](https://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html) are supported.
* Low cost (first million requests/month are free; $0.20 per 1 million requests thereafter).
* Lifecycle of Lambda function is a few milliseconds.

---
name: REST-API

## RESTful endpoints

Endpoint | Method | Authenticated? | Purpose | Consumer
--- | --- | --- | ---
`/job` | `POST` |  Yes | Start a new job. | Flask trigger app
`/job` | `GET` | No | Get information about running jobs. Can be constrained by parameters to display only jobs of interest. | SciScloud web app; anyone
`/stop_instances` | `POST` | Yes | Stop instances associated with a job. Only used by the final worker on a job. | Workers


A RESTful API is a good example of both a way that SciComp can
collaborate with labs, and a service that we can provide.

---

## Example output of `/job` endpoint (`GET` method):

```json
[
    {
        "bucket": "direct.typing",
        "hla_class": "2",
        "overall_status": "COMPLETE",
        "prefix": "2016-10-27-PT140",
        "session_key": "direct.typing-2016-10-27-PT140-class-1-161213-22:58:9397645206",
        "workers": [
            {
                "instance_id": "16f3230f6964b34915ebfa99dcc7f75a493e19aaff27350e4755a620d327cc75",
                "instance_index": 1,
                "samples": [
                 "K158154-Mi001716_S1_L001_R1_001.fastq.gz",
                 "K158154-Mi001716_S1_L001_R2_001.fastq.gz"
                ],
                "status": "COMPLETE"
            }
        ]
    }
]
```

---

## Does this project use Python 2 or Python 3?

TL;DR: Python 2.

* Python 3 was released in 2008. It is the future of the
  language. All new projects should use Python 3, and where
  possible, old projects should be ported to Python 3.
* This project was written in Python 2 when it came to SciComp.
  There were two roadblocks to porting the code to Python 3:
  * The BaseSpace API did not work in Python 3. I submitted
    a [pull request](https://github.com/basespace/basespace-python-sdk/pull/20)
    to fix this, which was accepted. This is no
    longer an issue as BaseSpace is no longer used
    in this project.
  * Lambda itself does not support Python 3. This is puzzling
    since it was launched in 2014. We have contacted AWS
    and requested Python 3 support.
* Since there are two separate codebases, and only one of them
  needs to run as a Lambda function, one of the codebases
  could theoretically be ported to Python 3. But this was
  deemed too confusing. So both codebases remain in Python 2 for now.
* Even though Python 3 is not used, we strive to write code
  that will work in both versions.

---
name: basespace

## About BaseSpace

[BaseSpace](https://basespace.illumina.com/home/index) is an
"app store" provided by Illumina (the maker of the sequencer)
used in the GeMS pipeline). Theoretically BaseSpace is nice because it allows users to use a wide-variety of third-party
"apps" against their data stored in the cloud.

In practice, its use is problematic, and it will no longer be
used in the GeMS pipeline because:

* Service was not reliable (downtime).
* Support was difficult to obtain.
* The API was not stable, and documentation was lacking
  or outdated.
* The presence of other apps was not thought to add
  value to the GeMS pipeline.
* No Python 3 support (until my [pull request](https://github.com/basespace/basespace-python-sdk/pull/23) was accepted).  




---
name: best-practices

## Best Practices

* Linters (pylint, flake8, pyflakes); [Atom](https://atom.io/) (editor) integration
* Unit tests
* Watching for [Anti-Patterns](https://docs.quantifiedcode.com/python-anti-patterns/)
* Continuous Integration (CI) for Java code with [CircleCI](https://circleci.com/). ([Travis](https://travis-ci.org/) is also very nice, and widely adopted, but CircleCI is smart enough to figure out how to build most projects without being told).
* Secrets Management with [CredStash](https://github.com/fugue/credstash) - a simple AWS-based solution, appropriate for this project since all parties share an AWS account. Other options might be [Chef Vault](https://github.com/chef/chef-vault) or [Hashicorp Vault](https://github.com/hashicorp/vault) (two Scicomp presentations on the latter are [here](https://fredhutch.github.io/svalbard-overview/#1) and [here](https://fredhutch.github.io/svalbard-intro/#1)).

<img src="img/linter.png" width="400" height="250" /> PyLint and Atom in action.

???
add one line about credstash - why we like it, who recommended it,
what problem it solves. other options are hashicorp vault, chef vault
(mention other stuff we are doing, ask mrg for input(?))

circleci - vs travis
circleci is nice because it figures out your type of project
travis is nice, widely adopted
expand on unit tests, why we used them, but why it's always a good idea too
underline keywords in long tables

mention why basespace is deprecated
 - reliability
 - support
 - api stablility & documentation issues
 - they did not need it

 in theory basespace is nice because ...
 but in practice not...

 keywords bind parts of diagrams to parts of text
 (in addition to numbers)


---



## The End

Slides can be found at:

[https://dtenenba.github.io/GeMS-HDC-Talk/](dtenenba.github.io/GeMS-HDC-Talk/)

The source for the slides can be found at:

[https://github.com/dtenenba/GeMS-HDC-Talk/](https://github.com/dtenenba/GeMS-HDC-Talk/)


### Acknowledgements

Thanks to the Geraghty Lab and the SciComp group.
