+++
date = "2012-04-27T21:12:00-05:00"
draft = false
title = "Daemonizing Go Programs (with a BSD-style rc.d example)"
author = "Andrew Gallant"
url = "golang-daemonize-bsd"
+++

Go, by its very nature, is multithreaded. This makes a traditional approach of
daemonizing Go programs by forking a bit difficult.

To get around this, you could try something as simple as backgrounding your Go
program and instructing it to [ignore the HUP
signal](http://en.wikipedia.org/wiki/Nohup):

<!--more-->

``` bash
nohup your-go-binary &
```

But what if your Go program is a web server that you need to be able to stop
and start?
(Particularly during development.)
It can quickly become a pain to use the above approach, as you'll have to look
up the process identifier each time you need to stop your server.
Moreover, using nohup isn't an ideal means to turn your program into a daemon,
since [it doesn't accomplish tasks commonly associated with
daemons](http://en.wikipedia.org/wiki/Daemon_\(computing\)#Creation), like
setting the root directory as the current working directory, setting the umask
to 0, and more.

In steps [daemonize](http://software.clapper.org/daemonize/), which runs any
command as a Unix daemon.
It automatically performs the aforementioned tasks and allows stdout and stderr
to be redirected to files of your choosing.
Here's a quick example usage:

``` bash
daemonize -o stdout.log -e stderr.log /absolute/path/to/go-program
```

While `daemonize` takes care of the nitty gritty details of becoming a daemon,
we still cannot start or stop our program as easily as we can with other common
daemons (like httpd, crond, cupsd, etc.).

In order to accomplish such a thing easily, it's usually convenient to mimmick
how other daemons are set up.

Since I use [Archlinux](http://www.archlinux.org/), my daemons
are organized in a BSD-style setup.
Namely, they are all located in `/etc/rc.d`.
Building off of the `/etc/rc.d/crond` daemon, I came up with the following for
the daemon powering this blog (located at `/etc/rc.d/blog-burntsushid`):

``` bash
#!/bin/bash

. /etc/rc.conf
. /etc/rc.d/functions

name=blog-burntsushi
logOut=/home/andrew/log/blog.stdout
logErr=/home/andrew/log/blog.stderr
full="/home/andrew/www/burntsushi.net/blog/blog"
cmd="/usr/sbin/daemonize -o $logOut -e $logErr $full"
user="andrew"

# Go environment setup.
export GOROOT=/opt/go
export GOPATH=/home/andrew/go/world:/home/andrew/go/me
export GOMAXPROCS=4

# You shouldn't have to edit below this line.
PID=$(pidof -o %PPID $full)

case "$1" in
start)
	stat_busy "Starting $name daemon"
	[[ -z "$PID" ]] && su $user -m -c "$cmd" &>/dev/null \
	&& { add_daemon $name; stat_done; } \
	|| { stat_fail; exit 1; }
	;;
stop)
	stat_busy "Stopping $name daemon"
	[[ -n "$PID" ]] && kill $PID &>/dev/null \
	&& { rm_daemon $name; stat_done; } \
	|| { stat_fail; exit 1; }
	;;
reload)
	stat_busy "Reloading $name daemon"
	[[ -n "$PID" ]] && kill -HUP $PID &>/dev/null \
	&& { stat_done; } \
	|| { stat_fail; exit 1; }
	;;
restart)
	$0 stop
	sleep 1
	$0 start
	;;
*)
	echo "usage: $0 {start|stop|restart|reload}"
	;;
esac
exit 0
```

Simply altering the environment variables at the top should be enough to adapt
it to your own purposes.
In particular, `$name` refers to the name of the daemon---it needn't correspond
to any actual file.
`$full` corresponds to the absolute path name of your Go binary.
(The path must be absolute because `daemonize` requires it.)
`$logOut` and `$logErr` correspond to the log files containing the stdout and
the stderr of your program.
`$cmd` corresponds to the full `daemonize` command.
`$user` is the name of the user that should run the daemon.
I've chosen to run my blog as myself for security purposes.
`$GOROOT`, `$GOPATH` and `$GOMAXPROCS` should be set according to your
Go environment.

Finally, the command is actually run using:

``` bash
su $user -m -c "$cmd"
```

Using `su` will run the daemon as the user you specified.
The `-m` switch tells `su` to use the current environment to run the command
in, which is required for the `$GO` variables to have any effect.

Your Go program can now be started, stopped or restarted like so:

``` bash
/etc/rc.d/blog-burntsushid start
/etc/rc.d/blog-burntsushid restart
/etc/rc.d/blog-burntsushid stop
```

The [Archlinux Wiki](https://wiki.archlinux.org/index.php/Main_Page) has more
information on [writing rc.d
scripts](https://wiki.archlinux.org/index.php/Writing_rc.d_scripts).

