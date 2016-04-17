# lldb-server
A RPC server interface for the LLDB Python bridge.

Principally the lldb debugger is usable as a python module but on many systems it is compiled against the system's default Python, if you want to use another Python version there will be incompatability problems.

Usually you don't want to have a debugger running in-process because if that crashes, all your Python code will crash with it. So we wrapped the lldb interface into an XMLRPC server (as that's in the standard lib of Python). So when the debugger crashes the worst thing that could happen would be an exception on the receiver side.

## Running the server process

Run the `lldb_server.py` with the Python installation that the `lldb` debugger was build against (usually it suffices to make `lldb_server.py` executable and start it directly).

`lldb_server.py` has two unnamed command line parameters:

- The first one is a path to the lldb python module. If you're using the open Source Version of Swift on OSX you'll need the following path:
    /Library/Developer/Toolchains/swift-latest.xctoolchain/System/Library/PrivateFrameworks/LLDB.framework/Resources/Python
  On Linux you will need something like:
    /home/user/swift/usr/lib/python2.7/site-packages
- The second one is the port to start the server on. It will always listen on the localhost loopback interface only.

## Using the server from witin Python

### Python 2.7

    import threading
    import xmlrpc
    from subprocess import Popen

    lldb_server_executable = "lldb_server.py"
    lldb_path = "/home/user/swift/usr/lib/python2.7/site-packages"
    port = 12345
    args = ['/usr/bin/python', lldb_server_executable, lldb_path, str(port)]

    p = Popen(args, cwd=path)
    threading.Thread(
        target=debugger_thread,
        name='debugger_thread',
        args=(p, port)
    ).start()

    def debugger_thread(p, port):
            lldb = xmlrpc.ServerProxy('http://localhost:' + str(port), allow_none=True)
            lldb.prepare(
                executable_path # executable to run
                [],             # cmdline args
                None,           # environment
                None,           # executable search path
                "."             # working dir
            )

            # Do whatever you want here

            lldb.shutdown_server()
            if p:
                p.wait()


### Python 3.3+

    import xmlrpc.client
    import threading
    from subprocess import Popen

    lldb_server_executable = "lldb_server.py"
    lldb_path = "/home/user/swift/usr/lib/python2.7/site-packages"
    port = 12345
    args = ['/usr/bin/python', lldb_server_executable, lldb_path, str(port)]

    p = Popen(args, cwd=path)
    threading.Thread(
        target=debugger_thread,
        name='debugger_thread',
        args=(p, port)
    ).start()

    def debugger_thread(p, port):
            lldb = xmlrpc.client.ServerProxy('http://localhost:' + str(port), allow_none=True)
            lldb.prepare(
                executable_path # executable to run
                [],             # cmdline args
                None,           # environment
                None,           # executable search path
                "."             # working dir
            )

            # Do whatever you want here

            lldb.shutdown_server()
            if p:
                p.wait()

## Server API

### Loading an executable

#### `prepare`

This command loads an executable and sets its startup parameters. The executable is started and immediately halted on the first instruction.

Parameters:
- 1: `executable`, string, the executable to load (make sure it has been compiled with debug symbols)
- 2: `params`, array of strings, command line parameters
- 3: `environment`, array of strings, environment variables to set (format like in C envp: `Name=Value`)
- 4: `path`, array of strings, directories to add to search path
- 5: `work_dir`, string, working directory

### Running, stopping, stepping

#### `start`

This starts or continues execution, it takes no parameters.

#### `pause`

This interrupts execution, it takes no parameters.

#### `step_into`

This makes a single source-code line step, stepping into function calls as it encounters one, it takes no parameters.

#### `step_over`

This makes a single source-code line step keeping the current execution context (e.g. stepping over function calls), it takes no parameters.

#### `step_out`

This continues running until the current stack frame is popped (e.g. a return call in a function, etc.), it takes no parameters.

#### `stop`

This stopps execution and exits the lldb server (target process is killed with `SIGKILL`), it takes no parameters.

### Shutting down

#### `shutdown_server`

This command has no parameters and will exit the server process, do not try to call any other RPC after this.

### LLDB Status

#### `get_status`

Fetch current debugger status, returns a string.
Common return values:

- `launching`: Executable is launching, libraries are loaded
- `running`: Executable is running
- `stopped` + reason: Executable has stopped, see below for reasons
- `stepping`: Currently executing a step command
- `crashed`: Executable has crashed
- `exited`: Executable has exited (shown only very briefly as the server exits on this state)

Uncommon return values (may happen when the user uses the console)

- `invalid`: Invalid status, internal error
- `unloaded`: Executable unloaded
- `connected`: Connected to external debug server (remote debugger)
- `attaching`: Attaching to external debug server or already running process
- `detached`: Detached from process or debug server
- `suspended`: Process that the debugger has been attached to has been suspended
- `unknown` + optional error code

Stop reasons, will be appended to `stopped` state with a comma:

- `breakpoint`: Stopped because of breakpoint
- `watchpoint`: Briefly interrupted because of watchpoint
- `signal`: Stopped because of UNIX signal, look for this state when waiting for the `prepare` command to finish
- `exception`: Exception was thrown (mostly used in ObjC and C++)
- `invalid`: Internal error
- `none`: No reason, probably the user called an interrupt command
- `trace`: Trace point hit
- `exec`: Target called `exec`
- `plan_complete`: User initiated action completed (status is used after step commands)
- `thread_exit`: Selected thread exited
- `instrumentation`: Instrumentation called a stop action (dtrace, etc.)

### Commandline IO

#### `get_stdout`

Fetches all data that has been buffered for `stdout`, takes no parameter, returns a string. If there has nothing been buffered, returns empty string.

#### `get_stderr`

Fetches all data that has been buffered for `stderr`, takes no parameter, returns a string. If there has nothing been buffered, returns empty string.

#### `push_stdin`

Send a string to `stdin` of target. Takes a string as parameter.

### Breakpoints

#### `get_breakpoints`

Fetch a list of all defined breakpoints. Takes no parameter.

Return value is an array of dictionary objects with the following keys:

- `file`: string, full path to the file the breakpoint is defined for
- `line`: integer, line number in the file
- `enabled`: boolean, enabled state of the breakpoint
- `condition`: string, condition when the breakpoint actually breaks
- `ignore_count`: integer, ignore the breakpoint n times before triggering
- `id`: integer, breakpoint id

#### `set_breakpoint`

Set a breakpoint for a specific file and line.
Parameters:

- 1: `filename`: string, full file path
- 2: `line_number`: integer, line number
- 3: `condition`: string, condition when the breakpoint triggers or `None`
- 4: `ignore_count`: integer, how often to ignore the breakpoint until triggering or `None`

#### `delete_breakpoint`

Delete a breakpoint.
Parameters:

- 1: `id`: integer, breakpoint id to delete

#### `enable_breakpoint`

Enable a breakpoint.
Parameters:

- 1: `id`: integer, breakpoint id to enable

#### `disable_breakpoint`

Disable a breakpoint.
Parameters:

- 1: `id`: integer, breakpoint id to disable

#### `disable_all_breakpoints`

Disable all breakpoints, takes no parameters.

#### `enable_all_breakpoints`

Enable all breakpoints, takes no parameters.

#### `delete_all_breakpoints`

Delete all breakpoints, takes no parameters.

#### `disable_breakpoints`

Temporarily disable all breakpoints, saves enabled state internally, takes no parameters.

#### `enable_breakpoints`

Re-Enable temporarily disabled breakpoints, pops the internal state, takes not parameters.

### Backtraces

#### `get_backtrace`

Get backtraces for all active threads, takes no parameters.

Return value is a dictionary which keys are the stringified thread IDs, values are dictionaries, union of thread info (see `get_threads`) and a `bt` component.
`bt` contains an array of stack frames, dictionaries on their own, with the following keys:

Resolvable frames (e.g. debug symbols available):

- `address`: string, stringified load address (to avoid integer overflows of Python's XMLRPC implementation)
- `module`: string, module name
- `function`: string, function name
- `file`: string, full path to file
- `line`: integer, line number
- `column`: integer, column in line if available, else 0
- `inlined`: boolean, has this function been inlined
- `arguments`: dictionary, key = argument name, value = argument value

Unresolvable stack frames (no debug symbols):

- `address`: string, stringified load address (to avoid integer overflows of Python's XMLRPC implementation)
- `module`: string, module name
- `symbol`: string, symbol name
- `offset`: string, stringified instruction offset into the symbol

#### `get_backtrace_for_selected_thread`

Get backtrace of currently selected thread, takes no parameters.

Return value is a dictionary. The dict is a union of thread info and a `bt` component. For format see `get_threads` and `get_backtrace`

### Threads

#### `get_threads`

Get thread info for all active threads, takes no parameters.

Return value is an array of dictionaries with the following keys:

- `id`, string, stringified thread id (to avoid integer overflows of Python's XMLRPC implementation)
- `index`, integer, thread index
- `name`, string, thread name or `None` if none was supplied
- `queue`, string, GCD queue if thread belongs to queue or `None`
- `stop_reason`: string, stop reason (see `get_status` for possible values) or `running`
- `num_frames`: integer, number of stack frames
- `selected`: boolean, True if this is the selected thread

#### `select_thread`

Select a thread ID as the current thread. Takes one parameter:

- 1: `id`: string, thread id as you got them from `get_threads`

#### `selected_thread`

Returns thread info for currently selected thread. For format of return value see `get_threads`.

### Variables

#### `get_arguments`

Get function arguments for a specific thread and stack frame. Parameters:

- 1: `thread_id`: string, the thread id you got from `get_threads`
- 2: `frame_index`: integer, index of the stack frame to query

Return value is a dictionary. Keys are the argument names (always strings), values the argument values (lldb description strings).

#### `get_local_variables`

Get local variables for a specific thread and stack frame. Parameters and return values are the same as in `get_arguments`.

#### `get_all_variables`

Get all variables (local, static, global) for a specific thread and stack frame. Parameters and return values are the same as in `get_arguments`.

### LLDB Console

#### `execute_lldb_command`

Execute a lldb command as if the user typed it into the console. Takes one parameter, a string with the command. Return value is a dictionary with the following keys:

- `succeeded`: boolean, was the command successful?
- `output`: string, command output or `None` if command failed
- `error`: string, error output or `None` if command succeeded
