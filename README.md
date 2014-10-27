Shake
=====

Shake is a build utility similar to Make and Rake. Both the `shake` utility and the Shakefiles supplied by the user are implemented as shell scripts. The implementation of `shake` is intended to be POSIX compliant.

Shake supports:
*   Dependencies between tasks.
*   Skipping a task based on the last modified dates of the file's dependencies.

Basic Usage
-----------

When invoked, Shake sources the Shakefile in the current directory. Any arguments passed to `shake` are passed to the `shake_main` function defined in the Shakefile. Based on those arguments, `shake_main` should place the name of the current task in `$shake_task` and execute any relevant commands. Shake provides two functions to help tie tasks together.

*   `depends_on [tasks...]` - Tasks supplied to depends_on will be run in the order given. If any of the dependencies fail the build will terminate. As an example the task `rebuild` might declare `depends_on clean build`.
*   `run [cmd]` - When running shell commands as part of a task, it's best to use this function. `run` will echo the command before evaluating it. If the command fails, `run` will display an error message and terminate the build.

Most Shakefiles will use the following template
```sh
#!/bin/sh

# Build Parameters
DEFAULT=build

shake_main() {
    shake_task=${1:-$DEFAULT}
    case "$shake_task" in
        # Task Defintions
    esac
}

```

More Complete Example
---------------------

Lets look at the sample project in `examples/c`. The project consists of
*   `hello.h`
*   `hello.c`
*   `main.c`

### Shakefile

The shebang isn't necessary, but may help your editor identify the file as a shell script.
```sh
#!/bin/sh
```
First we define the parameters of our build.
```sh
CC=c99
OBJECTS='main.o hello.o'
BIN=hello
```
Our Shakefile is required to implement a `shake_main` function.
```sh
shake_main() {
```
Inside of `shake_main`, we are required to set the `$shake_task` based on the parameters.
```sh
    shake_task=${1:-build}
```
The `depends_on` function is used to declare that the current task depends on others. `run` is a utility function provided to make debugging easier. It will echo the command before running it and will notify the use should the command fail.
```sh
    case "$shake_task" in
        build)
            depends_on $OBJECTS
            run $CC -o $BIN $OBJECTS
            ;;
```
Files are handled using splats in the case statement. If the task is a filename and that file exists, its last modified date will be compared against the files it depends on. The task will only be run if one of its dependencies is newer than the product.
```sh
        *.o)
            src=${shake_task%.o}.c
            depends_on "$src"
            run $CC -c -o "$shake_task" "$src"
            ;;
        clean)
            run rm -f *.o $BIN
            ;;
        rebuild)
            depends_on clean build
            ;;
        run)
            depends_on build
            run ./$BIN
            ;;
    esac
}
```

### Usage

```sh
$ shake build
[main.o] c99 -c -o main.o main.c
[hello.o] c99 -c -o hello.o hello.c
[build] c99 -o hello main.o hello.o
$ touch main.c # force main.o, and only main.o, to be rebuilt
$ shake build
[main.o] c99 -c -o main.o main.c
[build] c99 -o hello main.o hello.o
$ shake clean
[clean] rm -f *.o hello
```

Other shells
------------

By default Shake uses `/bin/sh` as its interpreter. It has been tested with `dash`, `posh`, and `bash`. Because it loads Shakefiles by sourcing them, the Shakefile will run under the same interpreter. The easiest way to change this is by editing `shake`'s shebang line.
