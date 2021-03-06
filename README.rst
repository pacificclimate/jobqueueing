==========================
PCIC PBS Job Queueing Tool
==========================

Purpose
=======

The PCIC PBS Job Queueing tool (hereafter, JQ) is a command-line tool for managing and recording the
submission of jobs to the PCIC compute nodes via PBS.

PBS (``qsub`` et al.) is a third-party tool that manages queueing and running of jobs on the PCIC compute nodes.
Even alone, PBS is very useful, but we have some additional requirements that are not directly met by it:

- Submit similar jobs (scripts) repeatedly with varying parameters (e.g, files to be processed).
- Record the fact and outcome of each such job permanently.

The solution is centered on a database that manages and records submissions to the PBS queue.
Since adding something to the database does not have to cause it to be submitted immediately
(nor is that necessarily desirable), JQ is effectively a "queue for a queue" in addition to its
function as a record-keeping service.

Installation
============

Easy
----

Create and activate a Python virtual environment::

    $ python3 -m venv venv
    $ venv/bin/activate
    (venv) $ pip install --upgrade pip

Install JQ from our pypi::

    (venv) $ pip install -i https://pypi.pacificclimate.org/simple/ jobqueueing


Geeky
-----

Clone the project repository from github::

    $ git clone git@github.com:pacificclimate/jobqueueing.git

As usual, it is best to install JQ in a Python virtual environment::

    $ cd jobqueueing
    $ python3 -m venv venv
    $ venv/bin/activate
    (venv) $ pip install --upgrade pip
    (venv) $ pip install .

Verify
------

This installs the ``jq`` command-line script. Verify installation by running::

    (venv) $ jq --help

Configuration
=============

There's little to configure in JQ, just a couple of environment variables and those all optional:

- ``JQ_DATABASE``: The Job Queueing database to run against. There are at present two standard databases
  set up on the gluster storage nodes.
  The location of these databases may change in future (but will remain on gluster).

  - ``/storage/data/projects/comp_support/climate_exporer_data_prep/climatological_means/jobqueue-prod.sqlite``
  - ``/storage/data/projects/comp_support/climate_exporer_data_prep/climatological_means/jobqueue-climdex.sqlite``
  - ``/storage/data/projects/comp_support/climate_exporer_data_prep/climatological_means/jobqueue-test.sqlite``

  The test database (``jobqueue-test.sqlite``) is exclusively for testing JQ, and should not be used for
  normal operations.

- ``JQ_PY_VENV``: The Python virtual environment in which to run *the script
  submitted to PBS*.

  This variable, if set, determines the default value for the
  ``jq add`` option ``-P`` / ``--py-venv``.

  (This is *not necessarily* the same as the virtual environment you are running
  JQ from!
  You *can* run JQ from the predefined environments below, because
  for convenience they have JQ installed as well as ``generate_climos``.)

There are a predefined environments suitable for JQ jobs at

- ``/storage/data/projects/comp_support/climate_exporer_data_prep/climatological_means/venv``
- ``/storage/data/projects/comp_support/climate_exporer_data_prep/climatological_means/venv_climdex``

which should be used in most cases.

In short, your JQ configuration should normally be::

    $ export JQ_DATABASE=/storage/data/projects/comp_support/climate_explorer_data_prep/climatological_means/jobqueue-prod.sqlite
    $ export JQ_PY_VENV=/storage/data/projects/comp_support/climate_explorer_data_prep/climatological_means/venv

Usage
=====

JQ commands are all of the form::

    jq [<global options>] <action> [<action options>]

To find out what the options and actions are, run::

    (venv) $ jq -h

To find out the options for specific actions, run::

    (venv) $ jq <action> -h

Actions overview
----------------

The actions available are, in alphabetical order:

=================== ============================================
Action              Effect
=================== ============================================
``add``             Add a file to the queue for processing with ``generate_climos``

``alter-params``    Update entries with generate_climos params and PBS params (but not status).
                    Updated entry must be in status NEW.

                    WARNING: ANY entry that partially matches the input filename is updated.

``hold``            Set entries in ``generate_climos`` queue from NEW to HOLD status

``unhold``          Set entries in ``generate_climos`` queue from HOLD to NEW status

``list``            List entries in ``generate_climos`` queue

``reset``           Reset the status of a queue entry

``script``          Write to stdout the script that will be or was submitted by ``jq submit`` for
                    a specified queue entry.

``submit``          Dequeue one or more ``generate_climos`` queue entries with NEW status,
                    and submit a PBS job for each, updating the queue entries accordingly.

``summarize``       Summarize entries in ``generate_climos`` queue (generate statistics)

``update-qstat``    Update ``generate_climos`` queue using PBS qstat
=================== ============================================


JQ workflow
===========

States
------

A JQ job is always in one of the following states (see "States and actions"
for details):

- ``NEW``
- ``HOLD``
- ``SUBMITTED``
- ``RUNNING``
- ``SUCCESS``
- ``ERROR``


Happy path workflow
-------------------

The typical, no-errors workflow is as follows:

=================== ======================= =================
State               Action                  Note
=================== ======================= =================
*no queue entry*      ``jq add``
``NEW``             ``jq submit``
``SUBMITTED``       ``jq update-qstat``
``RUNNING``         ``jq update-qstat``     This state entered only if update
                                            done while job actually running
``SUCCESS``                                 Job completed, data available
=================== ======================= =================

States and actions
------------------

This section describes the workflow for JQ. It describes the various states defined for JQ entries, and
the actions that cause transitions from one state to another. The general format is

``<JQ STATE>``
    description

    *Action*: Something that causes transition to another state
        | ``jq <action>``
        | --> ``<NEXT JQ STATE>``

**The JQ workflow is**:

[no queue entry]
    *Action*: Add to queue
        | ``jq add``
        | --> new JQ entry with status ``NEW``

``NEW``
    Job exists in JQ but has not been submitted to PBS.

    *Action*: Submit job to PBS
        | ``jq submit``
        | --> ``SUBMITTED``
    *Action*: Hold job (place in HOLD status) 
        | ``jq hold``
        | --> ``HOLD``

``HOLD``
    Job is not elegible for submission. Use this status to segregate jobs that
    for some reason you do not want to submit. (Submission is usually bulk
    and/or non-job-specific, in order of ``jq add`` -ing.)

    *Action*: Unhold (place in NEW status) 
        | ``jq unhold`` or ``jq reset``
        | --> ``NEW``


``SUBMITTED``
    Job has been submitted to PBS. Actual state of PBS job is unknown.
    The JQ state can be updated to reflect the PBS state by manual actions, see below.

    Now there is also a PBS status for the job, but it is not updated dynamically in JQ.

    *Action*: Update status while PBS job is running 
        | ``jq update-qstat``
        | --> ``RUNNING``
    *Action*: Update status after PBS job has terminated with success 
        | ``jq update-qstat``
        | --> ``SUCCESS``
    *Action*: Update status after PBS job has terminated with error 
        | ``jq update-qstat``
        | --> ``ERROR``
    *Action*: Reset (place job in NEW status).
        | Note: This makes the job elegible for submission again, but it
          removes any information about past submits.
        | ``jq reset``
        | --> ``NEW``

``RUNNING``
    Job has been submitted to PBS, and PBS job is known to be running.

    *Action*: Update status after PBS job has terminated with success 
        | ``jq update-qstat``
        | --> ``SUCCESS``
    *Action*: Update status after PBS job has terminated with error 
        | ``jq update-qstat``
        | --> ``ERROR``

``SUCCESS``
    Job has been submitted to PBS, and  PBS job completed normally.

    *Action*: Reset (place job in NEW status).
        | Note: This makes the job elegible for submission again, but it
          removes any information about past submits.
        | ``jq reset``
        | --> ``NEW``

``ERROR``
    Job has been submitted to PBS, and PBS job errored.

    *Action*: Reset (place job in NEW status).
        | Note: This makes the job elegible for submission again, but it
          removes any information about past submits.
        | ``jq reset``
        | --> ``NEW``

``jq submit`` and the work script
=================================

Most of JQ is just scaffolding to support what ``jq submit`` does,
which is to submit a script that causes the real work (generating
climatological means) to be done on a compute node.

TL;DR
-----

Here's where the data ends up:

* logs: ``<output dir>/logs/``
* output files: ``<output dir>/<pbs job num>/``
* temporary input files: ``$TMPDIR/climo/input``
* temporary output files: ``$TMPDIR/climo/output/<pbs job num>/``

Details
-------

The script submitted does the following things:

#. Set PBS options based on queue entry values:

   * processes per node (``ppn``)
   * virtual memory allocation (automatically computed from ppn)
   * walltime
   * ``stdout`` and ``stderr`` logs directed to ``logs/`` subdirectory of specified output directory
   * email notification
   * name ``generate_climos:<input filename>``

#. Set up the execution environment.

   * Load modules ``netcdf-bin``, ``cdo-bin``.
   * Activate the Python virtual environment specified for this queue entry.

#. Copy the input NetCDF file to ``$TMPDIR/climo/input``.

#. Set up the output directory structure in ``$TMPDIR``.

   * Base output directory for all outputs from this type of job is ``$TMPDIR/climo/output``.
   * Files containing climatological means generated in this particular PBS job are placed in
     ``$TMPDIR/climo/output/$pbs_job_num``, where
     ``pbs_job_num`` is the job number (e.g., ``1234``) extracted from the PBS job id for this job.

#. Generate climatological means.

   * Invoke ``generate_climos`` with the appropriate options and arguments.

#. Copy result file to final destination and remove temporary input file

   * Use ``rsync`` to update the final destination directory (specified for this queue entry)
     with the files created in the temporary directory. This causes the job id subdirectories
     to be replicated in the final destination directory, as well as the output files they contain.
   * Remove the temporary input file from ``$TMPDIR/climo/input``.
   * Since the output files are relatively small, we don't remove them from the temporary
     output directory, so that we have a fallback if something goes wrong with the ``rsync``.

Releasing
=========

To create a versioned release:

1. Increment ``__version__`` in ``setup.py``
2. Summarize the changes from the last release in ``NEWS.md``
3. Commit these changes, then tag the release::

    git add setup.py NEWS.md
    git commit -m"Bump to version x.x.x"
    git tag -a -m"x.x.x" x.x.x
    git push --follow-tags
