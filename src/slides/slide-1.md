class: center, middle
name: title

# GeMS Lambda Pipeline

## Dan Tenenbaum

<h3 id="display_time"></h3>

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

## How AWS Lambda is used in the pipeline

* Lambda lets you define "Lambda functions" that can be written in
  Python, Java, or Node.js. (Ours is in Python.)
* These functions can be triggered by numerous events.
* Using Amazon API Gateway, you can assign a URL to a Lambda function,
  and trigger the function by calling that URL.
* Using a tool called [Zappa](https://github.com/Miserlou/Zappa),
  you can write an ordinary [Flask](http://flask.pocoo.org/) application
  and deploy it as a Lambda function.
* In our case, the Flask application has both ordinary endpoints
  (for use by BaseSpace) and RESTful endpoints (for use by workers
  and interested clients such as webapps).
* Amazon has released its [Python Serverless Microframework for AWS](https://aws.amazon.com/blogs/developer/preview-the-python-serverless-microframework-for-aws/) (PSMA) which exposes a framework similar to,
  but different from, Flask. We prefer to use Flask directly,
  so we use Zappa.

---

## RESTful endpoints

Endpoint | Method | Authenticated? | Purpose
--- | --- | --- | ---
`/job` | `POST` |  Yes | Start a new job.
`/job` | `GET` | No | Get information about running jobs. Can be constrained by parameters to display only jobs of interest.
`/stop_instances` | `POST` | Yes | Stop instances associated with a job. Only used by the final worker on a job.

---

## Example output of `/job` endpoint:

```json
[
    {
        "bucket": "direct.typing",
        "hla_class": "2",
        "overall_status": "COMPLETE",
        "prefix": "2016-10-27-PT140",
        "session_key": "bwilkins2016sc-2016-08-12_meetingtest-class-1-161213-22:58:9397645206",
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
    to fix this, which was accepted.
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

## Best Practices

* Linters (pylint, flake8, pyflakes); [Atom](https://atom.io/) (editor) integration
* Unit tests
* Watching for [Anti-Patterns](https://docs.quantifiedcode.com/python-anti-patterns/)
* Continuous Integration (CI) for Java code with [CircleCI](https://circleci.com/)
* Secrets Management with [CredStash](https://github.com/fugue/credstash)


---

## The End

Slides can be found at:

[https://dtenenba.github.io/GeMS-HDC-Talk/](dtenenba.github.io/GeMS-HDC-Talk/)

The source for the slides can be found at:

[https://github.com/dtenenba/GeMS-HDC-Talk/](https://github.com/dtenenba/GeMS-HDC-Talk/)


### Acknowledgements

Thanks to the Geraghty Lab and the SciComp group.
