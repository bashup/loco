# LOcally-configured COmmands with `loco`

Sometimes, you need some project-specific commands or tasks, like those provided by `npm`, a `Makefile` or `Rakefile`... or maybe just a plain old shell function.

And sometimes, you want to invoke these commands from a *subdirectory* of your project directory, but still have them execute in the *root* of your project... possibly with some project-specific options.

So, `loco` is a simple shell script that looks for a project directory at or above your current working directory, reads some project-specific shell variables or functions, and then invokes the remainder of the command line in the matching directory with a customized environment.  For example, if you create the following `.loco` file in your project root:

```bash
loco.frob() { frobulate --cromulently -x 6 "$@"; }
loco.biz() { spizz --bizz "$@"; }
```

Then running `loco biz baz` in any subdirectory of your project will execute `spizz --bizz baz` in the project root.

The `.loco` file is sourced in the shell, so it can define functions, run programs, export variables...  whatever you need to do to make it work.  In addition, you can define certain special functions to change how loco works, define or override various defaults in `~/.locorc` and `/etc/loco/config`, or even rename, symlink, or extend the script to change the names it looks for.

For example, if you want to use `loco` with `docker-compose`, you might create a `doco` script like this and place it on your `PATH`:

```bash
#!/bin/bash -e
source "$(command -v loco)"

# Pass unrecognized subcommands to docker-compose
loco_exec() { docker-compose "$@"; }

# Do everything else the usual loco way
loco_main "$@"
```

When you run this new `doco` wrapper script, it will:

* look for site configuration in `/etc/doco/config`
* look for user configuration in `~/.docorc`, and
* look for local configuration in a `.doco`file in or above the current directory
* change to the directory where it was found
* expect its first argument to be a command defined as a `doco.commandname()` function
* pass its entire command line to `docker-compose` if there wasn't a matching function

You can also change any of these file naming conventions, behaviors, etc., as explained in the sections that follow.

But, if all you want to change is the file and function name search patterns, you don't even need a wrapper script: just rename or symlink `loco`, and the resulting script will use its own name to find its configuration files and commands.  (e.g., renaming or symlinking `loco` as `foo` will look for `/etc/foo/config`, `~/.foorc`, `.foo` and `foo.commandname()`.)

In addition, loco automatically defines the script name (`$LOCO_NAME`) as a function (if it doesn't already exist), so that recursive invocation of the script doesn't require a subshell or re-reading all the configuration files.  So in the example above, you could define a function like this:

```shell
doco.reup() {
    # Take all services down and back up again
    doco down && doco up -d
}
```

And rather than re-running the script multiple times, the nested `doco` commands will be directly dispatched to the appropriate functions (or `loco_exec()`).  Similarly, if you symlinked `loco` as `foo`, and there is no `foo` function already defined by the script, configuration, or project files, `loco_main` will define a `foo` function to prevent recursive invocation.

## Installation and Customization

To install `loco`, just copy it some place on your `PATH`, and start making configuration files or wrapper scripts.  (If you have [basher](https://github.com/basherpm/basher), you can just `basher install bashup/loco`.)

You can change the naming conventions for the site, user, and local configuration files by setting `LOCO_SITE_CONFIG`, `LOCO_RC`, and `LOCO_FILE` within a wrapper script after sourcing `loco`.  (It has to be *after*, since all `LOCO_` variables are unset during the sourcing.)

For example, if you wanted the `doco` wrapper script to get its site config from `/etc/docker/doco.conf`,  user-level config from `~/doco.conf` and project-level config from `docofile` files (instead of the default `~/.docorc` and `.doco`), you could add these lines to your wrapper script after it sources `loco`:

```bash
LOCO_SITE_CONFIG=/etc/docker/doco.conf
LOCO_RC=doco.conf
LOCO_FILE=docofile
```

(Note that `loco`'s environment variables and internal functions are *always* named `LOCO_` and `loco_` respectively, regardless of the active script name.  Only commands and config file names are based on `loco`'s script name or its wrapper script name.)

`LOCO_FILE`, by the way, can actually be a list of glob patterns: `LOCO_PROJECT` will be set to the absolute path of the first match.  So if our `doco` script set `LOCO_FILE="*.doco.md .doco"`, then each directory would first be checked for any file ending in `.doco.md` before being checked for a `.doco` file.  (Of course, the script would need to override `loco_loadproject()` to be able to handle all the different types of `LOCO_PROJECT` -- more on this below.)


## Defining Commands

By default, you define commands in your project, user, site, or global configuration files by defining functions prefixed with `loco`'s script name.  So if your script is named `fudge`, you might define `fudge.melt()` and `fudge.sweeten()` functions which would then be run in your project's root directories (as identified by `.fudge` files), when you type in `fudge melt` or `fudge sweeten`.

If, however, you type in `fudge mix`, and there is no `fudge.mix` function or command available, the `loco_exec()` function is called with `mix` and any remaining arguments.

The default implementation of `loco_exec()` emits an error message, but you can override it in any `loco` configuration file to do something different.  For example, you could pass the unrecognized command as a subcommand to some other program (such as `make`, `rake`, `gulp`, `docker`, etc.), as shown in the `docker-compose` example above.

### Exposed and/or Configurable Variables

There are a wide variety of variables you can set from your configuration files or wrapper scripts, and use in your functions or commands.  When `loco` is initially run or sourced, it unsets all of them, so it's best to source it at the start of your wrapper.)   Your  wrapper script can then set initial values for these variables, in which case the set value will be used in place of the defaults.  (The site, user, and project configuration files can also also set or override these variables, assuming they're loaded as shell scripts.)

After the default values of everything but `LOCO_PROJECT` and `LOCO_ROOT` have been set, your wrapper script's `loco_postconfig()` function will be called, if it exists.  This gives you a chance to *read* the end result of the configuration process prior to the main process execution.



| Variable           | Default Value                            | Notes                                    |
| ------------------ | ---------------------------------------- | ---------------------------------------- |
| `LOCO_SCRIPT`      | path to the script (may be relative to `LOCO_PWD`) | If `loco` is sourced, this will be the path to the sourcing script instead |
| `LOCO_COMMAND`     | `basename $LOCO_SCRIPT`                  | Used in `loco_usage()` message           |
| `LOCO_NAME`        | `$LOCO_COMMAND`                          | Base name for all configuration files, and command name prefix used by `loco_cmd`.  If a function of this name doesn't exist after configuration, `loco_main` will define it as a function alias for `loco_do`. |
| `LOCO_PWD`         | directory the script was invoked from    | Project directory search begins here     |
| `LOCO_SITE_CONFIG` | `/etc/$LOCO_NAME/config`                 | Full path of site-wide config file       |
| `LOCO_RC`          | `.${LOCO_NAME}rc`                        | User-level config file name              |
| `LOCO_USER_CONFIG` | `$HOME/$LOCO_RC`                         | User-level config file full path         |
| `LOCO_LOAD`        | `"source"`                               | Command or function used to read project-level config files |
| `LOCO_FILE`        | `.${LOCO_NAME}`                          | Space-separated list of globs matching project-level config files. |
| `LOCO_PROJECT`     | `$(loco_findproject "$@")`               | The found path of the project-level config file (not set until just before `loco_loadproject` is called) |
| `LOCO_ROOT`        | `$(dirname LOCO_PROJECT)`                | The project root directory, which `loco` will `cd` to before sourcing or reading`$LOCO_PROJECT` |

### Callable and/or Overrideable Functions

These functions can be called or overridden from your configuration files.  If you need to invoke the original implementation of one of these functions, you can do so by adding `_` to the start of the name.  For example, if you override `loco_do` , you can invoke `_loco_do` to execute its original behavior.

| Function           | Input(s)       | Default Results                          | Notes                                    |
| ------------------ | -------------- | ---------------------------------------- | ---------------------------------------- |
| `loco_main`        | *command line* | invokes `loco_config`, then locates the project root/config file, loads it, and runs the specified subcommand. | Can only be redefined from a wrapper script, since it's already running when config file(s) are loaded |
| `loco_config`      | *command line* | invokes `loco_preconfig` and `loco_postconfig` before and after calculating the default config file names and locations and loading them. | Can only be redefined from a wrapper script, since it's already running when config file(s) are loaded |
| `loco_site_config` | *filename*     | `source` *filename*                      | Override in a wrapper script to change how the `LOCO_SITE_CONFIG` file is loaded |
| `loco_user_config` | *filename*     | `source` *filename*                      | Override in a wrapper script or site config to change how the `LOCO_USER_CONFIG` file is loaded |
| `loco_preconfig`   | *command line* | no-op                                    | Override in a wrapper script to set initial values of `LOCO_*` variables, before the default values are calculated or configuration files are loaded |
| `loco_postconfig`  | *command line* | no-op                                    | Override in a wrapper script or user/site config to read or change the values of `LOCO_*` variables, after the default values have been calculated and any configuration files were loaded |
| `loco_findproject` | *command line* | `findup $LOCO_PWD $LOCO_FILE`            | Output the project file path on stdout.  Default implementation uses `findup` and emits an error if the project file isn't found.  Override this (anywhere but the project file) to change the way the project file is located. |
| `loco_findroot`    | *command line* | `dirname $LOCO_PROJECT`                  | Output the project root directory on stdout.  Default implementation just uses the directory the project file was found in.  Override this to change the way the project file is located. |
| `loco_loadproject` | *project-file* | `cd $LOCO_ROOT; $LOCO_SOURCE "$LOCO_PROJECT"` | Change to the project directory, and load the project file. |
| `loco_usage`       |                | Usage message to stderr; exit errorlevel 1 | Override to provide a more informative message |
| `loco_error`       | *message(s)*   | Outputs message to stderr, exit errorlevel 1 | Used by `loco_usage`                     |
| `loco_cmd`         | *commandname*  | `"$LOCO_NAME.$1"` (e.g. `loco.foo` for an input of `foo`) | Can be overridden to change the subcommand naming convention (e.g. to use a suffix instead of a prefix, `-` instead of `.`, or perhaps pointing to a subdirectory such as `node_modules/.bin`).  Empty output will trigger an error message and early termination. |
| `loco_exists`      | *commandname*  | Return truth if *commandname* is an existing function, alias, command, or shell builtin | Can be overridden to validate command existence some other way, but this is mostly useful to force fallback to `loco_exec()` even if a command exists.  (e.g. if you want to only recognize functions, not shell builtins or on-disk commands.) |
| `loco_exec`        | *command line* | Error message that command isn't recognized | Override this to pass unrecognized commands to a subcommand of, e.g. `rake`, `python setup.py` `docker`, `gulp`, etc. |
| `loco_do`          | *command line* | Translate first arg with `loco_cmd`, check existence with `loco_exists`, then directly execute or pass to `loco_exec` | It can be useful to invoke this when doing option parsing: just define functions like `loco.--arg()` that set a variable, then `shift` and `loco_do "$@"`. |



### Utility Functions

These functions can be used in your scripts, but **must not** be redefined, as they are also used internally by loco:

* `fn_exists` *functionname*  -- succeeds if *functionname* is the name of an existing bash function
* `fn_copy` *srcfunc* *newname* -- copies the body of *srcfunc* to a new function named *newname*; overwrites *newname* if it already exists.
* `findup` *dir* *globs...* -- beginning at *dir*, check for the existence of any files matching any of *globs*, and walk upwards to parent directories until a matching file is found or there's nowhere left to go.  On success, outputs the full path of the found file, otherwise nothing is output and failure is returned.
* `walkup`  *dir cmd args...* -- execute *cmd args...* *dir* for *dir* and every parent of *dir* until *cmd* returns success.  Note that this function does **not** change the current directory (though *cmd* is allowed to) and that when handling the *dir* argument you may need to address the root directory differently than other directories, since it is the only directory argument that will *end* in a slash as well as start with one.



## LICENSE

`loco` is copyright 2015-2017 PJ Eby, and MIT-licensed as follows:

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT OWNERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

