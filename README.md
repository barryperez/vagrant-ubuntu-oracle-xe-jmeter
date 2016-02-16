# JMeter results analysis - Oracle XE 11g on Ubuntu 12.04 using Vagrant

This project enables you to install Oracle 11g XE in a virtual machine running Ubuntu 12.04, using
[Vagrant] and [Puppet].

After installation of Oracle, an [external table] is created, to allow a JMeter result file to be dropped in for querying.

This is significantly faster than processing massive result files using scripts. On my machine, to produce overall stats for a file containing ~100M rows took < 6 minutes.

## Acknowledgements

The project [vagrant-ubuntu-oracle-xe] by Hilverd Reker was used as a starting point.

Thanks to Hilverd. His acknowledgements have also been included in full below:

The project was created based on the information in
[Installing Oracle 11g R2 Express Edition on Ubuntu 64-bit] by Manish Raj, and the GitHub repository
[vagrant-oracle-xe] by Stefan Glase. The former explains how to install Oracle XE 11g on Ubuntu
12.04, without explicitly providing a Vagrant or provisioner configuration. The latter has the same
purpose as this project but uses Ubuntu 11.10.

Thanks to André Kelpe, Brandon Gresham, Charles Walker, Chris Thompson, Jeff Caddel, Joe FitzGerald,
Justin Harringa, Mark Crossfield, Matthew Buckett, Richard Kolb, and Steven Hsu for various
contributions.

## Requirements

* You need to have [Vagrant] installed.
* The host machine probably needs at least 4 GB of RAM.
* As Oracle 11g XE is only available for 64-bit machines at the moment, the host machine needs to
  have a 64-bit architecture.
* You may need to [enable virtualization] manually.

## Installation

* Check out this project:

        git clone https://github.com/hilverd/vagrant-ubuntu-oracle-xe.git

* Install [vbguest]:

        vagrant plugin install vagrant-vbguest

* Download [Oracle Database 11g Express Edition] for Linux x64. Place the file
  `oracle-xe-11.2.0-1.0.x86_64.rpm.zip` in the directory `modules/oracle/files` of this
  project. (Alternatively, you could keep the zip file in some other location and make a hard link
  to it from `modules/oracle/files`.)

* Run `vagrant up` from the base directory of this project. The first time this will take a while -- up to 30 minutes on
  my machine. Please note that building the VM involves downloading an Ubuntu 12.04
  [base box](http://docs.vagrantup.com/v2/boxes.html) which is 323MB in size.

These steps are also shown in an [asciicast] made by Daekwon Kang:

[![asciicast](https://asciinema.org/a/8438.png)](https://asciinema.org/a/8438)

* Copy a JMeter result file in the directory `Data` and name it `jmeter.log`.

The file is expected to be in CSV format (header row shown):

    timeStamp;elapsed;label;responseCode;responseMessage;threadName;dataType;success;bytes;grpThreads;allThreads;Latency;SampleCount;ErrorCount;Hostname;"testid"

jmeter.properties values used:

    jmeter.save.saveservice.output_format=csv

    jmeter.save.saveservice.default_delimiter=;

    jmeter.save.saveservice.data_type=true
    jmeter.save.saveservice.label=true
    jmeter.save.saveservice.response_code=true

    jmeter.save.saveservice.response_message=true
    jmeter.save.saveservice.successful=true
    jmeter.save.saveservice.thread_name=true
    jmeter.save.saveservice.time=true
    jmeter.save.saveservice.subresults=true
    jmeter.save.saveservice.assertions=true
    jmeter.save.saveservice.latency=true
    jmeter.save.saveservice.bytes=true
    jmeter.save.saveservice.hostname=true
    jmeter.save.saveservice.thread_counts=true
    jmeter.save.saveservice.sample_count=true

    sample_variables=testid

For different formats of data, you will need to modify the `/modules/oracle/files/create_jmeter_tables.sql` file before running `Vagrant up`.

## Connecting

You should now be able to
[connect](http://www.oracle.com/technetwork/developer-tools/sql-developer/downloads/index.html) to
the new database at `localhost:1521/XE` as `system` with password `manager`. For example, if you
have `sqlplus` installed on the host machine you can do

    sqlplus system/manager@//localhost:1521/XE

To make sqlplus behave like other tools (history, arrow keys etc.) you can do this:

    rlwrap sqlplus system/manager@//localhost:1521/XE

You might need to add an entry to your `tnsnames.ora` file first:

    XE =
      (DESCRIPTION =
        (ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.1)(PORT = 1521))
        (CONNECT_DATA =
          (SERVER = DEDICATED)
          (SERVICE_NAME = XE)
        )
      )

## Querying the JMeter results

You should now be able to query the jmeter table:

Sample SQL (update start/end time as per your requirements):

    SELECT label,
      success,
      COUNT(*),
      MIN(elapsed) min,
      ROUND(AVG(elapsed),2) mean,
      percentile_disc (0.95) within GROUP (ORDER BY elapsed) AS perc95,
      MAX(elapsed) max,
      ROUND(stddev(elapsed),2) stdev
    FROM jmeter
      WHERE ts > ((to_date('11/02/2016 12:00:00', 'DD/MM/YYYY HH24:Mi:ss') - TO_DATE('1970-01-01', 'YYYY-MM-DD'))* 86.4)
      AND ts   < ((to_date('11/02/2016 13:00:00', 'DD/MM/YYYY HH24:Mi:ss') - TO_DATE('1970-01-01', 'YYYY-MM-DD'))* 86.4)
    GROUP BY label,
      success;

## Troubleshooting

### Errors when Unzipping

If you get an error containing `/usr/bin/unzip -o oracle-xe-11.2.0-1.0.x86_64.rpm.zip returned 2` during `vagrant up`, then the zip file you have downloaded is probably corrupted. This can be fixed by re-downloading, replacing the corrupted file, and running `vagrant reload --provision`.

### Memory

It is important to assign enough memory to the virtual machine, otherwise you will get an error

    ORA-00845: MEMORY_TARGET not supported on this system

during the configuration stage. In the `Vagrantfile` 1024 MB is assigned. Lower values may also work,
as long as (I believe) 2 GB of virtual memory is available for Oracle, swap is included in this
calculation.

### Concurrent Connections

If you want to raise the limit of the number of concurrent connections, say to 200, then according
to [How many connections can Oracle Express Edition (XE) handle?] you should run

    ALTER SYSTEM SET processes=200 scope=spfile

and restart the database.

## Alternatives

You may also want to consider a Docker-based solution such as
[docker-oracle-xe-11g](https://github.com/alexei-led/docker-oracle-xe-11g).

[Vagrant]: http://www.vagrantup.com/

[Puppet]: http://puppetlabs.com/

[Oracle Database 11g Express Edition]: http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html

[Oracle Database 11g EE Documentation]: http://docs.oracle.com/cd/E17781_01/index.htm

[Installing Oracle 11g R2 Express Edition on Ubuntu 64-bit]: http://meandmyubuntulinux.blogspot.co.uk/2012/05/installing-oracle-11g-r2-express.html

[vagrant-oracle-xe]: https://github.com/codescape/vagrant-oracle-xe

[vbguest]: https://github.com/dotless-de/vagrant-vbguest

[asciicast]: https://asciinema.org/a/8438

[How many connections can Oracle Express Edition (XE) handle?]: http://stackoverflow.com/questions/906541/how-many-connections-can-oracle-express-edition-xe-handle

[enable virtualization]: http://www.sysprobs.com/disable-enable-virtualization-technology-bios

[external table]: https://docs.oracle.com/cd/B28359_01/server.111/b28319/et_concepts.htm

[vagrant-ubuntu-oracle-xe]: https://github.com/hilverd/vagrant-ubuntu-oracle-xe