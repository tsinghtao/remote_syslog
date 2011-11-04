# remote_syslog Ruby daemon & sender

Lightweight Ruby daemon to tail one or more log files and transmit UDP syslog 
messages to a remote syslog host (centralized log aggregation).

remote_syslog generates UDP packets itself instead of depending on a system 
syslog daemon, so its configuration doesn't affect system-wide 
logging - syslog is just the transport.

Uses:

* collecting logs from servers & daemons which don't natively support syslog
* when reconfiguring the system logger is less convenient than a 
purpose-built daemon (e.g., automated app deployments)
* aggregating files not generated by daemons (e.g., package manager logs)

The library can also be used to generate one-off log messages from Ruby code.

Tested with the hosted log management service [Papertrail] and should work for 
transmitting to any syslog server.


## Installation

Install the gem, which includes a binary called "remote_syslog":

    $ [sudo] gem install remote_syslog

Optionally, create a log_files.yml with the log file paths to read and the 
host/port to log to (see examples/[log_files.yml.example][sample config]). These can also be 
specified as command-line arguments (below).


## Usage

    $ remote_syslog -h
    Usage:   remote_syslog [options] <path to add'l log 1> .. <path to add'l log n>

    Example: remote_syslog -c configs/logs.yml -p 12345 /var/log/mysqld.log

    Options:
        -c, --configfile PATH            Path to config (/etc/log_files.yml)
        -d, --dest-host HOSTNAME         Destination syslog hostname or IP (logs.papertrailapp.com)
        -p, --dest-port PORT             Destination syslog port (514)
        -D, --no-detach                  Don't daemonize and detach from the terminal
        -f, --facility FACILITY          Facility (user)
            --hostname HOST              Local hostname to send from
        -P, --pid-dir DIRECTORY          Directory to write .pid file in (/var/run/)
            --pid-file FILENAME          PID filename (<program name>.pid)
            --parse-syslog               Parse file as syslog-formatted file
        -s, --severity SEVERITY          Severity (notice)
            --strip-color                Strip color codes
            --tls                        Connect via TCP with TLS
        -h, --help                       Show this message
    

## Example

Typical:

    $ remote_syslog

Daemonize and collect messages from files listed in `./config/logs.yml` as 
well as the file `/var/log/mysqld.log`. Send to port `logs.papertrailapp.com:12345`:

    $ remote_syslog -c configs/logs.yml -p 12345 /var/log/mysqld.log

Stay attached to the terminal, look for and use `/etc/log_files.yml` if it
exists, write PID to `/tmp/remote_syslog.pid`, and send with facility local0
to `a.server.com:514`:

    $ remote_syslog -D -d a.server.com -f local0 -P /tmp /var/log/mysqld.log

remote_syslog will daemonize by default. A sample init file is in the gem as 
[remote_syslog.init.d]. You may be able to:

    $ cp examples/remote_syslog.init.d /etc/init.d/remote_syslog

## Sending messages securely ##

If the receiving system supports sending syslog over TCP with TLS, you can
pass the `--tls` option when running `remote_syslog`:

    $ remote_syslog --tls -p 1234 /var/log/mysqld.log


## Configuration

By default, the gem looks for a configuration in /etc/log_files.yml.

The gem comes with a [sample config].  Optionally:

    $ cp examples/log_files.yml.example /etc/log_files.yml
    
log_files.yml has filenames to log from (as an array) and hostname and port 
to log to (as a hash). Wildcards are supported using * and standard shell 
globbing. Filenames given on the command line are additive to those in 
the config file.

Only 1 destination server is supported; the command-line argument wins. 

    files: [/var/log/httpd/access_log, /var/log/httpd/error_log, /var/log/mysqld.log, /var/run/mysqld/mysqld-slow.log]
    destination:
      host: logs.papertrailapp.com
      port: 12345


## Advanced Configuration (Optional)

Here's an [advanced config] which uses all options.

### Override hostname

Provide `--hostname somehostname` or use the `hostname` configuration option:

    hostname: somehostname

### Multiple instances

Run multiple instances to support more than one message-specific file format 
or to specify unique syslog hostnames.

To do that, provide an alternate PID filename as a command-line option
to the additional instance(s). For example:

    --pid-file remote_syslog_2.pid

### Parse fields from log messages

Rarely needed. Usually only used when remote_syslog is watching files 
generated by syslogd (rather than by apps), like ``/var/log/messages``.

remote_syslog can parse the program and hostname from the log line. When one 
file contains logs from multiple programs (like with syslog), the log line 
may include text that is not part of the log message, like a timestamp, 
hostname, or program name. remote_syslog will extract those and use them in 
the corresponding syslog packet fields.

To do that, use the config file option `parse_fields` with the name of a 
format supported by remote_syslog, or your own regex. Included format names
are `syslog` and `rfc3339`. For example:

    parse_fields: syslog

The included `syslog` format uses the regex `(\w+ \d+ \S+) (\S+) ([^:]+): (.*)` 
to parse standard syslog lines like this:

    Jul 18 08:25:08 hostname programname[1234]: The log message

The included `rfc3339` format uses the regex `(\S+) (\S+) ([^: ]+):? (.*)` to 
parse syslog lines with high-precision RFC 3339 timestamps, like this:

    2011-07-16T08:25:08.651413-07:00 hostname programname[1234]: The log message

To parse a format other than those, provide your own regex. It should include 
4 backreferences to parse, in order: timestamp, system name, program name, 
message.

Match and return empty strings for any empty positions where the log line 
doesn't provide a value. For example, given the log message:

    something-meaningless The log message

One could use a regex to ignore "something-meaningless" (and not to extract
a program or hostname). To ignore that prefix and return 3 empty values 
then the log message, use parse_fields with this regex:

    parse_fields: "something-meaningless ()()()(.*)"

Per-file regexes are not supported. Run multiple instances with different 
config files.


## Reporting bugs

1. See whether the issue has already been reported: <https://github.com/papertrail/remote_syslog/issues/>
2. If you don't find one, create an issue with a repro case.


## Contributing

Once you've made your great commits:

1. [Fork][fk] remote_syslog
2. Create a topic branch - `git checkout -b my_branch`
3. Commit the changes without changing the Rakefile or other files unrelated to your enhancement.
4. Push to your branch - `git push origin my_branch`
5. Create a Pull Request or an [Issue][is] with a link to your branch
6. That's it!

[sample config]: https://github.com/papertrail/remote_syslog/blob/master/examples/log_files.yml.example
[remote_syslog.init.d]: https://github.com/papertrail/remote_syslog/blob/master/examples/remote_syslog.init.d
[advanced config]: https://github.com/papertrail/remote_syslog/blob/master/examples/log_files.yml.example.advanced
[fk]: http://help.github.com/forking/
[is]: https://github.com/papertrail/remote_syslog/issues/
[Papertrail]: http://papertrailapp.com/
