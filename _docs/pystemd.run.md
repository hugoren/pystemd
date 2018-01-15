# pystemd.run

> pystemd.run is inspired by systemd-run, but with a pythonic feel to it.

The key word here is _inspired_, pystemd.run can act and look like systemd-run
in many scenarios, but unlike pystemd, it's not a wrapper around the systemd-run
C calls, it's a reimplementation of the same ideas but in Python. With that
said, it is heavily inspired by the systemd-run source code, but it diverges from it
when needed to be more pythonic, or when systemd-run relies on systemd internals
that are not exposed as API.

One extra thing to notice is that pystemd.run is an API, not a command line tool,
so we will not assume that this is the only thing your program calls,
but we will assume that this just another building block for your program.

## Options:

* cmd: Array with the command to execute (absolute path only)
* name: Name of the unit, if not provided autogenerated.
* user: Username to execute the command, defaults to current user.
* user_mode: Equivalent to running `systemd-run --user`. Defaults to True
    if current user id not root (uid = 0).
* nice: Nice level to run the command.
* runtime_max_sec: set seconds before sending a sigterm to the process, if the
       service does not die nicely, it will send a sigkill.
* env: A dict with environmental variables.
* extra: If you know what you are doing, you can pass extra configuration
    settings to the start_transient_unit method.
* machine: Machine name to execute the command, by default we connect to
    the host's dbus.
* wait: wait for command completition before returning control, defaults
    to False.
* remain_after_exit: If True, the transient unit will remain after cmd
    has finish, also if true, this methods will return
    pystemd.systemd1.Unit object. defaults to False and this method
    returns None and the unit will be gone as soon as is done.
* pty: Set this variable to True if you want a pty to be created. if you
    pass a `machine`, the pty will be created in the machine. Setting
    this value will ignore whatever you set in pty_master and pty_path.
* pty_master: it has only meaning if you pass a pty_path also, this file
    descriptor will be used to foward redirection to `stdin` and `stdout`
    if no `stdin` or `stdout` is present, then this value does nothing.
* pty_path: Setting this value will pass this pty_path to the created
    process and will connect the process stdin, stdout and stderr to this
    pty. by itself it only ensure that your process has a real pty that
    can have ioctl operation over it. if you also pass a `pty_master`,
    `stdin` and `stdout` the pty forwars is handle for you.
* stdin: Specify a file descriptor for stdin, by default this is `None`
    and your unit will not have a stdin, you can specify it as
    `sys.stdin.fileno()`, or as a regular numer, e.g. `0`. If you set
    pty = True, or pass `pty_master` then this file descriptor will be
    read and forward to the pty.
* stdout: Specify a file descriptor for stdout, by default this is `None`
    and your unit will not have a stdout, you can specify it as
    `sys.stdout.fileno()`, or `open('/tmp/out', 'w').fileno()`, or a
    regular number, e.g. `1`. If you set pty = True, or pass `pty_master`
    then that pty will be read and forward to this file descriptor.
* stderr: Specify a file descriptor for stderr, by default this is `None`
    and your unit will not have a stderr, you can specify it as
    `sys.stderr.fileno()`, or `open('/tmp/err', 'w').fileno()`, or a
    regular number, e.g. `2`.


## Examples

  Usage examples (all run as root):

  1.- executes a ``/bin/sleep 42` in the background.

```python
>>> import pystemd.run
>>> pystemd.run([b'/bin/sleep', b'42'])
```

  2.- executes `/bin/sleep 42` but returns the units

```python
>>> import pystemd.run
>>> unit = pystemd.run([b'/bin/sleep', b'42'], remain_after_exit=True)
>>> unit
<pystemd.systemd1.unit.Unit at 0x7f8c460695c0>
>>> unit.Service.MainPID
66244
>>> # ... waiting 42 seconds later
>>> unit.Service.MainPID
0
```

  3.- executing `/bin/sleep 42` as other user

```python
>>> import pystemd.run
>>> unit = pystemd.run([b'/bin/sleep', b'42'], remain_after_exit=True,
    user=b'aleivag'
)
```

  4.- Multiple things: Executing `/bin/env`, on a machine named `miniarch`,
   watching the output of the command.

```python
>>> import pystemd.run, sys
>>> pystemd.run(
    [b'/bin/env'],
    machine=b'miniarch',
    wait=True,
    nice=3,
    stdout=sys.stdout.fileno(),
    env={b'SUPER_SECRET_PASSW': b'1234'}
    )
LANG=en_US.UTF-8
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
INVOCATION_ID=d7abcffb71ba419cbac448baa00cb495
SUPER_SECRET_PASSW=1234
```

  5.- Executing bash with a proper pty/tty, we specify stdin and stdout to do
  automatic redirection

```python
>>> import pystemd.run, sys
>>> pystemd.run([b'/bin/bash'], wait=True,
    stdout=sys.stdout,  stdin=sys.stdin, pty=True)

[root@localhost /]# ec<tab>
echo  ecpg  
[root@localhost /]# echo "hi all"
hi all
[root@localhost /]# sleep 30
^C
[root@localhost /]# sleep 60
^Z
[1]+  Stopped                 sleep 60
[root@localhost /]# jobs
[1]+  Stopped                 sleep 60
[root@localhost /]# exit
exit
>>>
```

## Extra notes:

1.- Please note that to use `pystemd.run`, you need to `import pystemd.run`,
importing pystemd and then calling `pystemd.run` will not work.

2.- Right now `pystemd.run` (the same as `pystemd`), does not accept string as
arguments but they have to be bytes, e.g. commands are not `['/bin/true']` they
are `[b'/bin/true']`

3.- stuff systemd-run does that pystemd.run does not (yet) does, but its on the
road-map:
    a) Run on a remote host.
    b) Run something different than a transient service, e.g. a timer.