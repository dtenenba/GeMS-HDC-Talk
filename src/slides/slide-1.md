class: center, middle
name: title

# GeMS Lambda Pipeline

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

## Architecture of the Pipeline (old version)

<!-- TODO: need Brandon/Wyatt to confirm my understanding of this -->

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
