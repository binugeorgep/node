# Child Process

    Stability: 2 - Stable

The `child_process` module provides the ability to spawn child processes in
a manner that is similar, but not identical, to [`popen(3)`][]. This capability
is primarily provided by the `child_process.spawn()` function:

```js
const spawn = require('child_process').spawn;
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.log(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

By default, pipes for `stdin`, `stdout` and `stderr` are established between
the parent Node.js process and the spawned child. It is possible to stream data
through these pipes in a non-blocking way. *Note, however, that some programs
use line-buffered I/O internally. While that does not affect Node.js, it can
mean that data sent to the child process may not be immediately consumed.*

The `child_process.spawn()` method spawns the child process asynchronously,
without blocking the Node.js event loop. The `child_process.spawnSync()`
function provides equivalent functionality in a synchronous manner that blocks
the event loop until the spawned process either exits of is terminated.

For convenience, the `child_process` module provides a handful of synchronous
and asynchronous alternatives to [`child_process.spawn()`][] and
[`child_process.spawnSync()`][].  *Note that each of these alternatives are
implemented on top of `child_process.spawn()` or `child_process.spawnSync()`.*

  * `child_process.exec()`: spawns a shell and runs a command within that shell,
    passing the `stdout` and `stderr` to a callback function when complete.
  * `child_process.execFile()`: similar to `child_process.exec()` except that
    it spawns the command directly without first spawning a shell.
  * `child_process.fork()`: spawns a new Node.js process and invokes a
    specified module with an IPC communication channel established that allows
    sending messages between parent and child.
  * `child_process.execSync()`: a synchronous version of
    `child_process.exec()` that *will* block the Node.js event loop.
  * `child_process.execFileSync()`: a synchronous version of
    `child_process.execFile()` that *will* block the Node.js event loop.

For certain use cases, such as automating shell scripts, the
[synchronous counterparts][] may be more convenient. In many cases, however,
the synchronous methods can have significant impact on performance due to
stalling the event loop while spawned processes complete.

## Asynchronous Process Creation

The `child_process.spawn()`, `child_process.fork()`, `child_process.exec()`,
and `child_process.execFile()` methods all follow the idiomatic asynchronous
programming pattern typical of other Node.js APIs.

Each of the methods returns a [`ChildProcess`][] instance. These objects
implement the Node.js [`EventEmitter`][] API, allowing the parent process to
register listener functions that are called when certain events occur during
the life cycle of the child process.

The `child_process.exec()` and `child_process.execFile()` methods additionally
allow for an optional `callback` function to be specified that is invoked
when the child process terminates.

### Spawning `.bat` and `.cmd` files on Windows

The importance of the distinction between `child_process.exec()` and
`child_process.execFile()` can vary based on platform. On Unix-type operating
systems (Unix, Linux, OSX) `child_process.execFile()` can be more efficient
because it does not spawn a shell. On Windows, however, `.bat` and `.cmd`
files are not executable on their own without a terminal, and therefore cannot
be launched using `child_process.execFile()`. When running on Windows, `.bat`
and `.cmd` files can be invoked using `child_process.spawn()` with the `shell`
option set, with `child_process.exec()`, or by spawning `cmd.exe` and passing
the `.bat` or `.cmd` file as an argument (which is what the `shell` option and
`child_process.exec()` do).

```js
// On Windows Only ...
const spawn = require('child_process').spawn;
const bat = spawn('cmd.exe', ['/c', 'my.bat']);

bat.stdout.on('data', (data) => {
  console.log(data);
});

bat.stderr.on('data', (data) => {
  console.log(data);
});

bat.on('exit', (code) => {
  console.log(`Child exited with code ${code}`);
});

// OR...
const exec = require('child_process').exec;
exec('my.bat', (err, stdout, stderr) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(stdout);
});
```

### child_process.exec(command[, options][, callback])

* `command` {String} The command to run, with space-separated arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `encoding` {String} (Default: 'utf8')
  * `shell` {String} Shell to execute the command with
    (Default: '/bin/sh' on UNIX, 'cmd.exe' on Windows,  The shell should
     understand the `-c` switch on UNIX or `/s /c` on Windows. On Windows,
     command line parsing should be compatible with `cmd.exe`.)
  * `timeout` {Number} (Default: 0)
  * `maxBuffer` {Number} largest amount of data (in bytes) allowed on stdout or
    stderr - if exceeded child process is killed (Default: `200*1024`)
  * `killSignal` {String} (Default: 'SIGTERM')
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* `callback` {Function} called with the output when process terminates
  * `error` {Error}
  * `stdout` {Buffer}
  * `stderr` {Buffer}
* Return: {ChildProcess}

Spawns a shell then executes the `command` within that shell, buffering any
generated output.

```js
const exec = require('child_process').exec;
const child = exec('cat *.js bad_file | wc -l',
  (error, stdout, stderr) => {
    console.log(`stdout: ${stdout}`);
    console.log(`stderr: ${stderr}`);
    if (error !== null) {
      console.log(`exec error: ${error}`);
    }
});
```

If a `callback` function is provided, it is called with the arguments
`(error, stdout, stderr)`. On success, `error` will be `null`.  On error,
`error` will be an instance of [`Error`][]. The `error.code` property will be
the exit code of the child process while `error.signal` will be set to the
signal that terminated the process. Any exit code other than `0` is considered
to be an error.

The `options` argument may be passed as the second argument to customize how
the process is spawned. The default options are:

```js
{
  encoding: 'utf8',
  timeout: 0,
  maxBuffer: 200*1024,
  killSignal: 'SIGTERM',
  cwd: null,
  env: null
}
```

If `timeout` is greater than `0`, the parent will send the the signal
identified by the `killSignal` property (the default is `'SIGTERM'`) if the
child runs longer than `timeout` milliseconds.

The `maxBuffer` option specifies the largest amount of data (in bytes) allowed
on stdout or stderr - if this value is exceeded then the child process is
terminated.

*Note: Unlike the `exec()` POSIX system call, `child_process.exec()` does not
replace the existing process and uses a shell to execute the command.*

### child_process.execFile(file[, args][, options][, callback])

* `file` {String} The name or path of the executable file to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `encoding` {String} (Default: 'utf8')
  * `timeout` {Number} (Default: 0)
  * `maxBuffer` {Number} largest amount of data (in bytes) allowed on stdout or
    stderr - if exceeded child process is killed (Default: 200\*1024)
  * `killSignal` {String} (Default: 'SIGTERM')
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* `callback` {Function} called with the output when process terminates
  * `error` {Error}
  * `stdout` {Buffer}
  * `stderr` {Buffer}
* Return: {ChildProcess}

The `child_process.execFile()` function is similar to [`child_process.exec()`][]
except that it does not spawn a shell. Rather, the specified executable `file`
is spawned directly as a new process making it slightly more efficient than
[`child_process.exec()`][].

The same options as `child_process.exec()` are supported. Since a shell is not
spawned, behaviors such as I/O redirection and file globbing are not supported.

```js
const execFile = require('child_process').execFile;
const child = execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    throw error;
  }
  console.log(stdout);
});
```

### child_process.fork(modulePath[, args][, options])

* `modulePath` {String} The module to run in the child
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `execPath` {String} Executable used to create the child process
  * `execArgv` {Array} List of string arguments passed to the executable
    (Default: `process.execArgv`)
  * `silent` {Boolean} If true, stdin, stdout, and stderr of the child will be
    piped to the parent, otherwise they will be inherited from the parent, see
    the `'pipe'` and `'inherit'` options for [`child_process.spawn()`][]'s
    [`stdio`][] for more details (default is false)
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* Return: {ChildProcess}

The `child_process.fork()` method is a special case of
[`child_process.spawn()`][] used specifically to spawn new Node.js processes.
Like `child_process.spawn()`, a `ChildProcess` object is returned. The returned
`ChildProcess` will have an additional communication channel built-in that
allows messages to be passed back and forth between the parent and child. See
[`ChildProcess#send()`][] for details.

It is important to keep in mind that spawned Node.js child processes are
independent of the parent with exception of the IPC communication channel
that is established between the two. Each process has it's own memory, with
their own V8 instances. Because of the additional resource allocations
required, spawning a large number of child Node.js processes is not
recommended.

By default, `child_process.fork()` will spawn new Node.js instances using the
`process.execPath` of the parent process. The `execPath` property in the
`options` object allows for an alternative execution path to be used.

Node.js processes launched with a custom `execPath` will communicate with the
parent process using the file descriptor (fd) identified using the
environment variable `NODE_CHANNEL_FD` on the child process. The input and
output on this fd is expected to be line delimited JSON objects.

*Note: Unlike the `fork()` POSIX system call, [`child_process.fork()`][] does
not clone the current process.*

### child_process.spawn(command[, args][, options])

* `command` {String} The command to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `stdio` {Array|String} Child's stdio configuration. (See
    [`options.stdio`][])
  * `detached` {Boolean} Prepare child to run independently of its parent
    process. Specific behavior depends on the platform, see
    [`options.detached`][])
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `shell` {Boolean|String} If `true`, runs `command` inside of a shell. Uses
    '/bin/sh' on UNIX, and 'cmd.exe' on Windows. A different shell can be
    specified as a string. The shell should understand the `-c` switch on UNIX,
    or `/s /c` on Windows. Defaults to `false` (no shell).
* return: {ChildProcess}

The `child_process.spawn()` method spawns a new process using the given
`command`, with command line arguments in `args`. If omitted, `args` defaults
to an empty array.

A third argument may be used to specify additional options, with these defaults:

```js
{
  cwd: undefined,
  env: process.env
}
```

Use `cwd` to specify the working directory from which the process is spawned.
If not given, the default is to inherit the current working directory.

Use `env` to specify environment variables that will be visible to the new
process, the default is `process.env`.

Example of running `ls -lh /usr`, capturing `stdout`, `stderr`, and the
exit code:

```js
const spawn = require('child_process').spawn;
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.log(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```


Example: A very elaborate way to run 'ps ax | grep ssh'

```js
const spawn = require('child_process').spawn;
const ps = spawn('ps', ['ax']);
const grep = spawn('grep', ['ssh']);

ps.stdout.on('data', (data) => {
  grep.stdin.write(data);
});

ps.stderr.on('data', (data) => {
  console.log(`ps stderr: ${data}`);
});

ps.on('close', (code) => {
  if (code !== 0) {
    console.log(`ps process exited with code ${code}`);
  }
  grep.stdin.end();
});

grep.stdout.on('data', (data) => {
  console.log(`${data}`);
});

grep.stderr.on('data', (data) => {
  console.log(`grep stderr: ${data}`);
});

grep.on('close', (code) => {
  if (code !== 0) {
    console.log(`grep process exited with code ${code}`);
  }
});
```


Example of checking for failed exec:

```js
const spawn = require('child_process').spawn;
const child = spawn('bad_command');

child.on('error', (err) => {
  console.log('Failed to start child process.');
});
```

#### options.detached

On Windows, setting `options.detached` to `true` makes it possible for the
child process to continue running after the parent exits. The child will have
its own console window. *Once enabled for a child process, it cannot be
disabled*.

On non-Windows platforms, if `options.detached` is set to `true`, the child
process will be made the leader of a new process group and session. Note that
child processes may continue running after the parent exits regardless of
whether they are detached or not.  See `setsid(2)` for more information.

By default, the parent will wait for the detached child to exit. To prevent
the parent from waiting for a given `child`, use the `child.unref()` method.
Doing so will cause the parent's event loop to not include the child in its
reference count, allowing the parent to exit independently of the child, unless
there is an established IPC channel between the child and parent.

Example of detaching a long-running process and redirecting its output to a
file:

```js
const fs = require('fs');
const spawn = require('child_process').spawn;
const out = fs.openSync('./out.log', 'a');
const err = fs.openSync('./out.log', 'a');

const child = spawn('prg', [], {
 detached: true,
 stdio: [ 'ignore', out, err ]
});

child.unref();
```

When using the `detached` option to start a long-running process, the process
will not stay running in the background after the parent exits unless it is
provided with a `stdio` configuration that is not connected to the parent.
If the parent's `stdio` is inherited, the child will remain attached to the
controlling terminal.

#### options.stdio

The `options.stdio` option is used to configure the pipes that are established
between the parent and child process. By default, the child's stdin, stdout,
and stderr are redirected to corresponding `child.stdin`, `child.stdout`, and
`child.stderr` streams on the `ChildProcess` object. This is equivalent to
setting the `options.stdio` equal to `['pipe', 'pipe', 'pipe']`.

For convenience, `options.stdio` may be one of the following strings:

* `'pipe'` - equivalent to `['pipe', 'pipe', 'pipe']` (the default)
* `'ignore'` - equivalent to `['ignore', 'ignore', 'ignore']`
* `'inherit'` - equivalent to `[process.stdin, process.stdout, process.stderr]`
   or `[0,1,2]`

Otherwise, the value of `option.stdio` is an array where each index corresponds
to an fd in the child. The fds 0, 1, and 2 correspond to stdin, stdout,
and stderr, respectively. Additional fds can be specified to create additional
pipes between the parent and child. The value is one of the following:

1. `'pipe'` - Create a pipe between the child process and the parent process.
   The parent end of the pipe is exposed to the parent as a property on the
   `child_process` object as `ChildProcess.stdio[fd]`. Pipes created for
   fds 0 - 2 are also available as ChildProcess.stdin, ChildProcess.stdout
   and ChildProcess.stderr, respectively.
2. `'ipc'` - Create an IPC channel for passing messages/file descriptors
   between parent and child. A ChildProcess may have at most *one* IPC stdio
   file descriptor. Setting this option enables the ChildProcess.send() method.
   If the child writes JSON messages to this file descriptor, the
   `ChildProcess.on('message')` event handler will be triggered in the parent.
   If the child is a Node.js process, the presence of an IPC channel will enable
   `process.send()`, `process.disconnect()`, `process.on('disconnect')`, and
   `process.on('message')` within the child.
3. `'ignore'` - Instructs Node.js to ignore the fd in the child. While Node.js
   will always open fds 0 - 2 for the processes it spawns, setting the fd to
   `'ignore'` will cause Node.js to open `/dev/null` and attach it to the
   child's fd.
4. `Stream` object - Share a readable or writable stream that refers to a tty,
   file, socket, or a pipe with the child process. The stream's underlying
   file descriptor is duplicated in the child process to the fd that
   corresponds to the index in the `stdio` array. Note that the stream must
   have an underlying descriptor (file streams do not until the `'open'`
   event has occurred).
5. Positive integer - The integer value is interpreted as a file descriptor
   that is is currently open in the parent process. It is shared with the child
   process, similar to how `Stream` objects can be shared.
6. `null`, `undefined` - Use default value. For stdio fds 0, 1 and 2 (in other
   words, stdin, stdout, and stderr) a pipe is created. For fd 3 and up, the
   default is `'ignore'`.

Example:

```js
const spawn = require('child_process').spawn;

// Child will use parent's stdios
spawn('prg', [], { stdio: 'inherit' });

// Spawn child sharing only stderr
spawn('prg', [], { stdio: ['pipe', 'pipe', process.stderr] });

// Open an extra fd=4, to interact with programs presenting a
// startd-style interface.
spawn('prg', [], { stdio: ['pipe', null, null, null, 'pipe'] });
```

*It is worth noting that when an IPC channel is established between the
parent and child processes, and the child is a Node.js process, the child
is launched with the IPC channel unreferenced (using `unref()`) until the
child registers an event handler for the `process.on('disconnected')` event.
This allows the child to exit normally without the process being held open
by the open IPC channel.*

See also: [`child_process.exec()`][] and [`child_process.fork()`][]

## Synchronous Process Creation

The `child_process.spawnSync()`, `child_process.execSync()`, and
`child_process.execFileSync()` methods are **synchronous** and **WILL** block
the Node.js event loop, pausing execution of any additional code until the
spawned process exits.

Blocking calls like these are mostly useful for simplifying general purpose
scripting tasks and for simplifying the loading/processing of application
configuration at startup.

### child_process.execFileSync(file[, args][, options])

* `file` {String} The name or path of the executable file to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration. (Default: 'pipe')
    - `stderr` by default will be output to the parent process' stderr unless
      `stdio` is specified
  * `env` {Object} Environment key-value pairs
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number} largest amount of data (in bytes) allowed on stdout or
    stderr - if exceeded child process is killed
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
* return: {Buffer|String} The stdout from the command

The `child_process.execFileSync()` method is generally identical to
`child_process.execFile()` with the exception that the method will not return
until the child process has fully closed. When a timeout has been encountered
and `killSignal` is sent, the method won't return until the process has
completely exited. *Note that if the child process intercepts and handles
the `SIGTERM` signal and does not exit, the parent process will still wait
until the child process has exited.*

If the process times out, or has a non-zero exit code, this method ***will***
throw.  The [`Error`][] object will contain the entire result from
[`child_process.spawnSync()`][]

### child_process.execSync(command[, options])

* `command` {String} The command to run
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration. (Default: 'pipe')
    - `stderr` by default will be output to the parent process' stderr unless
      `stdio` is specified
  * `env` {Object} Environment key-value pairs
  * `shell` {String} Shell to execute the command with
    (Default: '/bin/sh' on UNIX, 'cmd.exe' on Windows,  The shell should
     understand the `-c` switch on UNIX or `/s /c` on Windows. On Windows,
     command line parsing should be compatible with `cmd.exe`.)
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number} largest amount of data (in bytes) allowed on stdout or
    stderr - if exceeded child process is killed
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
* return: {Buffer|String} The stdout from the command

The `child_process.execSync()` method is generally identical to
`child_process.exec()` with the exception that the method will not return until
the child process has fully closed. When a timeout has been encountered and
`killSignal` is sent, the method won't return until the process has completely
exited. *Note that if  the child process intercepts and handles the `SIGTERM`
signal and doesn't exit, the parent process will wait until the child
process has exited.*

If the process times out, or has a non-zero exit code, this method ***will***
throw.  The [`Error`][] object will contain the entire result from
[`child_process.spawnSync()`][]

### child_process.spawnSync(command[, args][, options])

* `command` {String} The command to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration.
  * `env` {Object} Environment key-value pairs
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number} largest amount of data (in bytes) allowed on stdout or
    stderr - if exceeded child process is killed
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
  * `shell` {Boolean|String} If `true`, runs `command` inside of a shell. Uses
    '/bin/sh' on UNIX, and 'cmd.exe' on Windows. A different shell can be
    specified as a string. The shell should understand the `-c` switch on UNIX,
    or `/s /c` on Windows. Defaults to `false` (no shell).
* return: {Object}
  * `pid` {Number} Pid of the child process
  * `output` {Array} Array of results from stdio output
  * `stdout` {Buffer|String} The contents of `output[1]`
  * `stderr` {Buffer|String} The contents of `output[2]`
  * `status` {Number} The exit code of the child process
  * `signal` {String} The signal used to kill the child process
  * `error` {Error} The error object if the child process failed or timed out

The `child_process.spawnSync()` method is generally identical to
`child_process.spawn()` with the exception that the function will not return
until the child process has fully closed. When a timeout has been encountered
and `killSignal` is sent, the method won't return until the process has
completely exited. Note that if the process intercepts and handles the
`SIGTERM` signal and doesn't exit, the parent process will wait until the child
process has exited.

## Class: ChildProcess

Instances of the `ChildProcess` class are [`EventEmitters`][] that represent
spawned child processes.

Instances of `ChildProcess` are not intended to be created directly. Rather,
use the [`child_process.spawn()`][], [`child_process.exec()`][],
[`child_process.execFile()`][], or [`child_process.fork()`][] methods to create
instances of `ChildProcess`.

### Event: 'close'

* `code` {Number} the exit code if the child exited on its own.
* `signal` {String} the signal by which the child process was terminated.

The `'close'` event is emitted when the stdio streams of a child process have
been closed. This is distinct from the `'exit'` event, since multiple
processes might share the same stdio streams.

### Event: 'disconnect'

The `'disconnect'` event is emitted after calling the
`ChildProcess.disconnect()` method in the parent or child process. After
disconnecting it is no longer possible to send or receive messages, and the
`ChildProcess.connected` property is false.

### Event:  'error'

* `err` {Error} the error.

The `'error'` event is emitted whenever:

1. The process could not be spawned, or
2. The process could not be killed, or
3. Sending a message to the child process failed.

Note that the `'exit'` event may or may not fire after an error has occurred.
If you are listening to both the `'exit'` and `'error'` events, it is important
to guard against accidentally invoking handler functions multiple times.

See also [`ChildProcess#kill()`][] and [`ChildProcess#send()`][].

### Event:  'exit'

* `code` {Number} the exit code if the child exited on its own.
* `signal` {String} the signal by which the child process was terminated.

The `'exit'` event is emitted after the child process ends. If the process
exited, `code` is the final exit code of the process, otherwise `null`. If the
process terminated due to receipt of a signal, `signal` is the string name of
the signal, otherwise `null`. One of the two will always be non-null.

Note that when the `'exit'` event is triggered, child process stdio streams
might still be open.

Also, note that Node.js establishes signal handlers for `SIGINT` and
`SIGTERM` and Node.js processes will not terminate immediately due to receipt
of those signals. Rather, Node.js will perform a sequence of cleanup actions
and then will re-raise the handled signal.

See `waitpid(2)`.

### Event: 'message'

* `message` {Object} a parsed JSON object or primitive value.
* `sendHandle` {Handle} a [`net.Socket`][] or [`net.Server`][] object, or
  undefined.

The `'message'` event is triggered when a child process uses `process.send()`
to send messages.

### child.connected

* {Boolean} Set to false after `.disconnect` is called

The `child.connected` property indicates whether it is still possible to send
and receive messages from a child process. When `child.connected` is false, it
is no longer possible to send or receive messages.

### child.disconnect()

Closes the IPC channel between parent and child, allowing the child to exit
gracefully once there are no other connections keeping it alive. After calling
this method the `child.connected` and `process.connected` properties in both
the parent and child (respectively) will be set to `false`, and it will be no
longer possible to pass messages between the processes.

The `'disconnect'` event will be emitted when there are no messages in the
process of being received. This will most often be triggered immediately after
calling `child.disconnect()`.

Note that when the child process is a Node.js instance (e.g. spawned using
[`child_process.fork()`]), the `process.disconnect()` method can be invoked
within the child process to close the IPC channel as well.

### child.kill([signal])

* `signal` {String}

The `child.kill()` methods sends a signal to the child process. If no argument
is given, the process will be sent the `'SIGTERM'` signal. See `signal(7)` for
a list of available signals.

```js
const spawn = require('child_process').spawn;
const grep = spawn('grep', ['ssh']);

grep.on('close', (code, signal) => {
  console.log(
    `child process terminated due to receipt of signal ${signal}`);
});

// Send SIGHUP to process
grep.kill('SIGHUP');
```

The `ChildProcess` object may emit an `'error'` event if the signal cannot be
delivered. Sending a signal to a child process that has already exited is not
an error but may have unforeseen consequences. Specifically, if the process
identifier (PID) has been reassigned to another process, the signal will be
delivered to that process instead which can have unexpected results.

Note that while the function is called `kill`, the signal delivered to the
child process may not actually terminate the process.

See `kill(2)`

### child.pid

* {Number} Integer

Returns the process identifier (PID) of the child process.

Example:

```js
const spawn = require('child_process').spawn;
const grep = spawn('grep', ['ssh']);

console.log(`Spawned child pid: ${grep.pid}`);
grep.stdin.end();
```

### child.send(message[, sendHandle[, options]][, callback])

* `message` {Object}
* `sendHandle` {Handle}
* `options` {Object}
* `callback` {Function}
* Return: {Boolean}

When an IPC channel has been established between the parent and child (
i.e. when using [`child_process.fork()`][]), the `child.send()` method can be
used to send messages to the child process. When the child process is a Node.js
instance, these messages can be received via the `process.on('message')` event.

For example, in the parent script:

```js
const cp = require('child_process');
const n = cp.fork(`${__dirname}/sub.js`);

n.on('message', (m) => {
  console.log('PARENT got message:', m);
});

n.send({ hello: 'world' });
```

And then the child script, `'sub.js'` might look like this:

```js
process.on('message', (m) => {
  console.log('CHILD got message:', m);
});

process.send({ foo: 'bar' });
```

Child Node.js processes will have a `process.send()` method of their own that
allows the child to send messages back to the parent.

There is a special case when sending a `{cmd: 'NODE_foo'}` message. All messages
containing a `NODE_` prefix in its `cmd` property are considered to be reserved
for use within Node.js core and will not be emitted in the child's
`process.on('message')` event. Rather, such messages are emitted using the
`process.on('internalMessage')` event and are consumed internally by Node.js.
Applications should avoid using such messages or listening for
`'internalMessage'` events as it is subject to change without notice.

The optional `sendHandle` argument that may be passed to `child.send()` is for
passing a TCP server or socket object to the child process. The child will
receive the object as the second argument passed to the callback function
registered on the `process.on('message')` event.

The `options` argument, if present, is an object used to parameterize the
sending of certain types of handles. `options` supports the following
properties:

  * `keepOpen` - A Boolean value that can be used when passing instances of
    `net.Socket`. When `true`, the socket is kept open in the sending process.
    Defaults to `false`.

The optional `callback` is a function that is invoked after the message is
sent but before the child may have received it.  The function is called with a
single argument: `null` on success, or an [`Error`][] object on failure.

If no `callback` function is provided and the message cannot be sent, an
`'error'` event will be emitted by the `ChildProcess` object. This can happen,
for instance, when the child process has already exited.

`child.send()` will return `false` if the channel has closed or when the
backlog of unsent messages exceeds a threshold that makes it unwise to send
more. Otherwise, the method returns `true`. The `callback` function can be
used to implement flow control.

#### Example: sending a server object

The `sendHandle` argument can be used, for instance, to pass the handle of
a TSCP server object to the child process as illustrated in the example below:

```js
const child = require('child_process').fork('child.js');

// Open up the server object and send the handle.
const server = require('net').createServer();
server.on('connection', (socket) => {
  socket.end('handled by parent');
});
server.listen(1337, () => {
  child.send('server', server);
});
```

The child would then receive the server object as:

```js
process.on('message', (m, server) => {
  if (m === 'server') {
    server.on('connection', (socket) => {
      socket.end('handled by child');
    });
  }
});
```

Once the server is now shared between the parent and child, some connections
can be handled by the parent and some by the child.

While the example above uses a server created using the `net` module, `dgram`
module servers use exactly the same workflow with the exceptions of listening on
a `'message'` event instead of `'connection'` and using `server.bind` instead of
`server.listen`. This is, however, currently only supported on UNIX platforms.

#### Example: sending a socket object

Similarly, the `sendHandler` argument can be used to pass the handle of a
socket to the child process. The example below spawns two children that each
handle connections with "normal" or "special" priority:

```js
const normal = require('child_process').fork('child.js', ['normal']);
const special = require('child_process').fork('child.js', ['special']);

// Open up the server and send sockets to child
const server = require('net').createServer();
server.on('connection', (socket) => {

  // If this is special priority
  if (socket.remoteAddress === '74.125.127.100') {
    special.send('socket', socket);
    return;
  }
  // This is normal priority
  normal.send('socket', socket);
});
server.listen(1337);
```

The `child.js` would receive the socket handle as the second argument passed
to the event callback function:

```js
process.on('message', (m, socket) => {
  if (m === 'socket') {
    socket.end(`Request handled with ${process.argv[2]} priority`);
  }
});
```

Once a socket has been passed to a child, the parent is no longer capable of
tracking when the socket is destroyed. To indicate this, the `.connections`
property becomes `null`. It is recommended not to use `.maxConnections` when
this occurs.

### child.stderr

* {Stream}

A `Readable Stream` that represents the child process's `stderr`.

If the child was spawned with `stdio[2]` set to anything other than `'pipe'`,
then this will be `undefined`.

`child.stderr` is an alias for `child.stdio[2]`. Both properties will refer to
the same value.

### child.stdin

* {Stream}

A `Writable Stream` that represents the child process's `stdin`.

*Note that if a child process waits to read all of its input, the child will not
continue until this stream has been closed via `end()`.*

If the child was spawned with `stdio[0]` set to anything other than `'pipe'`,
then this will be `undefined`.

`child.stdin` is an alias for `child.stdio[0]`. Both properties will refer to
the same value.

### child.stdio

* {Array}

A sparse array of pipes to the child process, corresponding with positions in
the [`stdio`][] option passed to [`child_process.spawn()`][] that have been set
to the value `'pipe'`. Note that `child.stdio[0]`, `child.stdio[1]`, and
`child.stdio[2]` are also available as `child.stdin`, `child.stdout`, and
`child.stderr`, respectively.

In the following example, only the child's fd `1` (stdout) is configured as a
pipe, so only the parent's `child.stdio[1]` is a stream, all other values in
the array are `null`.

```js
const assert = require('assert');
const fs = require('fs');
const child_process = require('child_process');

const child = child_process.spawn('ls', {
    stdio: [
      0, // Use parents stdin for child
      'pipe', // Pipe child's stdout to parent
      fs.openSync('err.out', 'w') // Direct child's stderr to a file
    ]
});

assert.equal(child.stdio[0], null);
assert.equal(child.stdio[0], child.stdin);

assert(child.stdout);
assert.equal(child.stdio[1], child.stdout);

assert.equal(child.stdio[2], null);
assert.equal(child.stdio[2], child.stderr);
```

### child.stdout

* {Stream}

A `Readable Stream` that represents the child process's `stdout`.

If the child was spawned with `stdio[1]` set to anything other than `'pipe'`,
then this will be `undefined`.

`child.stdout` is an alias for `child.stdio[1]`. Both properties will refer
to the same value.

[`popen(3)`]: http://linux.die.net/man/3/popen
[`ChildProcess`]: #child_process_child_process
[`child_process.exec()`]: #child_process_child_process_exec_command_options_callback
[`child_process.execFile()`]: #child_process_child_process_execfile_file_args_options_callback
[`child_process.fork()`]: #child_process_child_process_fork_modulepath_args_options
[`child_process.spawn()`]: #child_process_child_process_spawn_command_args_options
[`child_process.spawnSync()`]: #child_process_child_process_spawnsync_command_args_options
[`ChildProcess#kill()`]: #child_process_child_kill_signal
[`ChildProcess#send()`]: #child_process_child_send_message_sendhandle_callback
[`Error`]: errors.html#errors_class_error
[`EventEmitter`]: events.html#events_class_events_eventemitter
[`EventEmitters`]: events.html#events_class_events_eventemitter
[`net.Server`]: net.html#net_class_net_server
[`net.Socket`]: net.html#net_class_net_socket
[`options.detached`]: #child_process_options_detached
[`options.stdio`]: #child_process_options_stdio
[`stdio`]: #child_process_options_stdio
[synchronous counterparts]: #child_process_synchronous_process_creation
