AreWeFastYet
============

AreWeFastYet is a set of tools used for benchmarking the major browser's
JavaScript virtual machines against each other, as well as reporting the
results on a website as insightful graphs showing the evolution of performance
over time.

A public instance is hosted by Mozilla and running at
[https://arewefastyet.com](https://arewefastyet.com).

Components
==========

Slave:

1. Builder: A python driver (build.py) that can create shell builds of
   spidermonkey/jsc/v8.
2. Downloader: A python driver (download.py) that can download browser builds
   of Firefox.
3. Executor: (execute.py) is a python script that executes one or multiple
   benchmarks on one or more builds.

Site:

1. Database: MySQL database that stores statistics.
2. Collector: Hidden PHP script on the webserver, where stats get sent.
   (UPDATE.php in below diagram)
3. Processor: Python aggregator that builds JSON data from the DB. (update.py
   in below diagram)
4. Website: Static HTML as the frontpage, that queries JSON via XHR.
5. Command center: Sends commands to the slaves on what to execute. (In
   construction.)

Components (2) and (4) must be on the same webserver, otherwise timestamps
might not be computed correctly.

Keep in mind, most of this documentation is for posterity. AWFY was never
intended to be a drag-and-drop all-in-one released product, so the procedures
and scripts may be pretty rough.

System Diagram
==============
<!--
    Image source at
    https://docs.google.com/drawings/d/1TlzFOMz4oxKYSD_hHqe-fL2wpAA5WwhEq62KJp0TytI/edit?usp=sharing
    To edit it, 'make a copy' (from file menu) and edit that.
-->
![Alt text](/docs/awfy_system_diagram.png "System Diagram")


Benchmark locally
=================

1. Fetch the repo
2. Create a (shell) or retrieve a (browser) build to benchmark
  * Creating a build:
    ```
    cd slave
    python build.py -s mozilla
    ```

  * Pull a build:

    ```
    cd slave
    python download.py http://archive.mozilla.org/pub/mozilla.org/firefox/tinderbox-builds/mozilla-inbound-linux/latest/
    ```

3. Benchmark

    ```
    python execute.p -b remote.octane -b remote.kraken
    ```

Installation
============

Database
--------
Put `/server/awfy-server.config` in `/etc`, and edit it to point at your
database. Afterwards just run `php migrate.php` to create the tables and run
the migrations.  (Note: sometimes the database layout changes a bit. After
pulling it is recommanded to run `php migrate.php` again. That will incremental
adjust the database to the new layout, transforming existing entries.)

Data Collector
--------------
Drop `website/UPDATE.PHP` and `website/internals.php` somewhere, and rename
`UPDATE.PHP` to something secret. Make sure you don't have directory listings
enabled.

Slave DNS Config
----------------
Before running the benchmarks, add these lines to your `/etc/hosts` file and
flush DNS cache. These host addresses are used by `benchmarks_remote.py`. This
configuration is needed for all the slaves that use the remote or local
benchmarks. Only the shell benchmarks don't need it.

```
# Subdomains for AWFY
127.0.0.1   dromaeo.localhost
127.0.0.1   kraken.localhost
127.0.0.1   octane.localhost
127.0.0.1   massive.localhost
127.0.0.1   jetstream.localhost
127.0.0.1   speedometer.localhost
127.0.0.1   speedometer.localhost
127.0.0.1   sunspider.localhost
127.0.0.1   browsermark.local
```


Benchmark Computers
-------------------

In development...

Data Processor
--------------
Put `awfy-server.config` in `/etc`, and edit it to point at your database and
website/data folder. Then put `update.py` in a cronjob. It will dump files
where appropriate. AWFY.com does this every 15min. It is not safe to run two
instance at once. A sample wrapper script is provided as `run-update.sh`.

update.py generates various JSON files:

1. "raw" and "metadata" files cache database queries from run to run, so we
   don't have to make expensive database queries.
2. "aggregate" files are used for the front page.
3. "condensed" files are used for one level of zooming, so users don't have to
   download the raw data set right away.

The metadata and raw JSON files are updated as needed. The aggregate and
condensed files are always regenerated from the raw data.

There is also a `monitor.py` script provided in the server folder. You can run
this regularly to send e-mails for benchmarking machines that haven't sent
results in a certain amount of time (this time is specified in
`awfy-server.config`). It will send e-mail through the local SMTP server, using
the "contact" field for each machine in the database. This field should be a
comma-delimited list of e-mail addresses (i.e. "egg@yam.com,bob@egg.com").

Website
-------
Put the files somewhere. Currently php is needed for `data.php`, which pulls the
data from the correct location. You just need to update config file
(`/etc/awfy-server.config`) to refer the 'data' folder that contains the
json/js files dumped by update.py.

Don't forget to replace the default machine number in website/awfy.js, which is
the one that will show up in the first place. Note that AWFY's flot is slightly
modified, so it might not work to just replace it with upstream flot.
