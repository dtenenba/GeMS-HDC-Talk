class: center, middle
name: title

# GeMS Lambda Pipeline

## Dan Tenenbaum

### (figure out how to add today's date dynamically)
???

These are presenter notes (that's what the ??? indicates).

See the talk at (production URL)
https://dtenenba.github.io/GeMS-HDC-Talk/
To build locally, do:

    gulp

To deploy to production url above (probably have to commit first?):

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

The result of that work is what we are presenting today.

---

## What is the GeMS Pipeline?

High-level overview presented by a member of the Geraghty Lab
(Brandon or Wyatt).

(Hope to add content contributed by them...)

* Purpose
* What does GeMS stand for?
* Input (fasta files from illumina sequencer)
* Output (a report that can be viewed in GeMS-UI) (screenshots?)
* HLA Typing
---

## Architecture of the Pipeline (old version), part 1

<!-- TODO: need Brandon/Wyatt to confirm my understanding of this -->

![old pipeline part 1](img/old_pipeline_1.jpg)


---

## Architecture of the Pipeline (old version), part 2

### Inside a single worker

![old pipeline part 2](img/old_pipeline_2.jpg)

---

class: small

## Old Pipeline Components

&nbsp;

| Component | Description |
| --- | --- |
| [Illumina MiSeq Sequencer](https://www.illumina.com/systems/sequencing-platforms/miseq.html) | Produces initial data (in fastq.gz format) |
| [Amazon S3](https://aws.amazon.com/s3/) | File storage at all stages of the pipline |
| [Amazon SimpleDB](https://aws.amazon.com/simpledb/) | Deprecated key-value store, used for queueing and storing job metadata. Not really a queuing system. |
| [BaseSpace](https://basespace.illumina.com/home/index) | Illumina's app gateway and file storage space, one of two ways to kick off the pipeline. |
| [ScisCloud Web Application](https://sciscloud.herokuapp.com/) | A second way to kick off the pipeline. |
| Queueing Server | An always-on [EC2](https://aws.amazon.com/ec2/) instance listening for new jobs, either via a Python web server (for jobs from BaseSpace) or SSH connections (for jobs from ScisCloud) |
| Worker Pool | A group of stopped EC2 instances, some of which are started when jobs come in. |
| Python Code (private repository) | The glue that holds everything together and orchestrates the steps in the pipeline. |
| Java Code  (private repository) | Code that performs the actual HLA typing  |
| [cross_match](http://www.bioinformatics.org/wiki/Phred_Phrap_Consed_Cross_match) (C program) | Third-party tool that compares DNA sequence sets. |



---

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


---

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

## Much more stuff to be added here...

---

## Best Practices

* Linters (pylint, flake8, pyflakes)
* Unit tests
* Watching for [Anti-Patterns](https://docs.quantifiedcode.com/python-anti-patterns/)
* Continuous Integration (CI) for Java code


---

## The End

Slides can be found .here.

The source for the slides is .here.


### Acknowledgements

Thanks to the Geraghty Lab and the SciComp group.
