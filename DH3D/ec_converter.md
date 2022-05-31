## Editor
#### subprocess.Popen
```python
p = subprocess.Popen(
	args,
	stdout=subprocess.PIPE,
	stderr=subprocess.PIPE,
	shell=shell
	cwd=cwd,
	env=env,
)
p = wait
```
subprocess.Popen()用于开启一端进程，其中Popen可用参数如下
```python
""" Execute a child program in a new process.
    For a complete description of the arguments see the Python documentation.
    Arguments:
      args: A string, or a sequence of program arguments.
      bufsize: supplied as the buffering argument to the open() function when creating the stdin/stdout/stderr pipe file objects
      executable: A replacement program to execute.
      stdin, stdout and stderr: These specify the executed programs' standard input, standard output and standard error file handles, respectively.
      preexec_fn: (POSIX only) An object to be called in the child process just before the child is executed.
      close_fds: Controls closing or inheriting of file descriptors.
      shell: If true, the command will be executed through the shell.
      cwd: Sets the current directory before the child is executed.
      env: Defines the environment variables for the new process.
      text: If true, decode stdin, stdout and stderr using the given encoding (if set) or the system default otherwise.
      universal_newlines: Alias of text, provided for backwards compatibility.
      startupinfo and creationflags (Windows only)
      restore_signals (POSIX only)
      start_new_session (POSIX only)
      group (POSIX only)
      extra_groups (POSIX only)
      user (POSIX only)
      umask (POSIX only)
      pass_fds (POSIX only)
      encoding and errors: Text mode encoding and error handling to use for file objects stdin, stdout and stderr.
    Attributes:
      stdin, stdout, stderr, pid, returncode

    """
```
```Popen.wait()```
等待进程执行至结束
```Popen.communicate()```
等待进程执行至结束，并且返回`stdout`和`stderr`
#### env
其中`env`为环境变量，需要注意的是编码形式
`env['PYTHONIOENCODING'] = 'UFT8'`
或者在json中配置
```json
"env": {
	"PYTHONIOENCODING": "utf-8"
}
```

## CommonWaitLogDialog
### showWaitOutputDialog()
### closeWaitOutputDialog()
## Logger
### handler
handler是log的处理器，注意不要直接实例化`Handler`，这个类用来派生其他更有用的子类。但是，子类的`__init__()`方法需要调用`Handler.__init__()`

**`emit(record)`**
执行实际记录给定日志记录所需的操作。这个版本应由子类实现，因此这里直接引发`NotImplementedError`异常