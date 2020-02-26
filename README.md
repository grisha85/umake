UMake
=====
**Blazing Fast. Sub scecond Modification detection. Few seconds for cached compilation**

Overview
--------
UMake is a build system that building your projects.  
influenced by [`tup`](http://gittup.org/tup/). 

* local cache - disk cache
* remote cache - minio
* auto dependency discovery using strace
* simple configuration language

Install
-------
```
https://github.com/grisha85/umake.git
pip3 install umake
```
- strace and bash are `umake` dependencies

Minimal UMakefile
-----------------
```
# macros
!c(includes, flags) : gcc -g -Wall -fPIC -c {filename} $includes $flags -o {target} > {dir}/{noext}.o
!app-c(includes, flags, libs) : gcc -g {filename} $includes $flags $libs -o {target}

# compile c
$includes = -Iinclude
$cflags = -O0
:foreach src/*.c > !c(includes, $cflags)
: *.o > !app-c( , , ) > myapp
```
add this `UMakfile` to your `root` directory of your project. and run `umake --no-remote-cache`

How UMake works
---------------
* loading compilation graph (DAG) from previous build (graph is empty when built first time)
  
* scannig filesytem using the loaded graph to check for changes

  * `modified` - if file was modified on filesystem it marked as modified on the graph
  * `deleted` - if file was deleted from the filesystem, successor target deleted as well from the filesystem and the graph
    

* parsing UMakefile and creating new commands or updating the existing

  * `deleted` - if command deleted, targets of this command deleted as well from both graph and filesystem
  * `updated` - if command is updated, UMake will handle it as `delete` and `create`. so, target of the old command will be deleted and new target will be created when graph will be executed
* executing the graph in parallel 

  * `auto dependency detection` - updating the graph with accessed files by parsing strace logs
  * `cache` - saving to cache. more details: [Cache System](#cache-system)
* saving the build graph 

Note: by automatically deleting `targets` when no longer needed (either `command` is delete or source file was the deleted for this `target`) UMake implementing `clean` by design. So `clean` no longer need to be mainained.

UMakefile
---------
## Rule `:`

A signle `command` is generated for this rule

`:` source | manual-deps `>` cmd `>` target

`manual-deps` - targets the this tule depends on, in order to keep a correct build order  
`cmd` - bash command  
`target` - the output of the command  
  
`{filename}` - full path filename of the source `/my/path/filename.a`  
`{dir}` - directory of the source `/my/path/`  
`{noext}` - filename without extension `filename`  
`{target}` - expanded target `helloworld.a`  

Example:
```
: *.o > gcc {filename} -o {target} > helloworld.a
```

#### Recursive Source `**`
recursice deps are support
```
root\
  a\
    a.a\
      a.a.a
      a.a.b
      a.b.a
    a.b\
      a.b.a
      a.b.b
  b\
    b
```
* `root/**` -> (`a.a.a`, `a.a.b`, `b`)
* `root/a/**/*.b` -> (`a.a.b`, `a.b.b`)

#### Manual Dependency `|`
In order to maintain the correct order of the build (that is done in parallel), there are use cases that manual depndecy is needed to be provided. for example: if there are `generated headers` that used later by other `command` to generate anoter target. 


## Rule `:foreach`
Same as `:` but will create `command` for each `source` (that match the pettern *.o in the example above) file that will be found on the filesystem

## Macro `!`
Macros are expanded immediatlly (like `#define X "hello"` in c/cpp)  
Macros can accept parameters

Example:
```
!c(includes, flags) : gcc -g -O2 -Wall -fPIC -c {filename} $includes $flags -o {target} > {dir}/{noext}.o  
```
#### Default values
`Macro` supports defualt values, by default it `""`:
```
!c(includes, flags=-O3) : gcc -g -O2 -Wall -fPIC -c {filename} $includes $flags -o {target} > {dir}/{noext}.o  
```
now `!c` can be called as following
```
!c(-Iinclude)       # includes = -Iinclude, flags=-O3
!c(-Iinclude, -O0)  # includes = -Iinclude, flags=-O0
!c()                # includes = "", flags=-O3
```
## Const `$`
Consts are like macros, and can be used to parametrize calls to macros
Example:
```
$libs = -lpthread
!so($libs)
```

## Config `[<config_item>:<config_value>]`
Configs allow to configure and changing umake execution.

#### `workdir`
Default: \<root>

Changing current working directory. After changing the working directory all `relative paths` are relative to the new working dir. `Absoulte paths` are relative to the `root` (the directory where UMakefile exists).
Relative path `my_dir_a/my_dir_b` will be evaluated as `<workdir>/my_dir_a/my_dir_b`. However `/my_dir_a/my_dir_b` will be evaluated as `<root>/my_dir_a/my_dir_b` regardless what working dir is.

The following rules are similar:

```
: src/packages/a > gcc > src/packages/b
```
```
[workdir:src/packages]
: a > gcc > b
```
Return to root
```
[workdir:/]
```

#### `variant`

Defult: "default"

The ability to generate diffrent variants from the same sources. For example: debug/release compilations. variant `terminated` with a `newline`
```
# varaint is terminated with newline
[variant:default]
$cflags = -O3

[variant:debug]
$cflags = -O0

: my.c > !c($cflags) > my.o
```
now compile with `umake` for default variant 
```
umake
```
or
```
umake --variant debug
```
for `debug` variant.

#### `include`
Default: -

include another `UMakefile` into the current one.
```
[include:somedir/umakefile]
```
will open and parse `somedir/umakefile` in the current working dir context.
# Cache System
Targets are being cached after creation, and checked if the target is in cache just before executing a `command`. There are two types of cache that UMake is using local(filesystem) and remote (minio). 

## How Cache works
### On Save
* `sha1` of the target sources (those that were generated from UMakefile) are being calculated and `sha1` of the `command` itself. All dependecies files (also those that were auto detected) Saved to `md-<calculated_hash>` file.
* `sha1` of all dependecies are calculated and the just created target is saved to `<all_dependecies_hash>/<target_name_hash>`
### On Load
* `sha1` of the target sources (those that were generated from UMakefile) are being calculated and `sha1` of the `command` itself. Reading `md-<calculated_hash>` for all the file dependecies
* calculating `sha1` of all of the target dependecies (from the files system) and copying `<all_dependecies_hash>/<target_name_hash>` to the project directory as it was generated by the `command`

## Local Cache
The local cache is stored in `~/.umake/build-cache`. 

## Remote Cache
TBD
# Arguments

```
usage: umake [-h] [--details] [--json JSON_FILE] [--no-remote-cache]
             [--no-local-cache]
             [target]

positional arguments:
  target             target path

optional arguments:
  -h, --help         show this help message and exit
  --details          details about the target
  --json JSON_FILE   output as json
  --no-remote-cache  don't use remote cache
  --no-local-cache   don't use local cache

```

