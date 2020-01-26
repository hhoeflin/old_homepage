---
layout: post
title: "Cluster computation in Python"
date: 2020-01-25
categories: python, cluster
description: "Easy way to do cluster computation in python from scratch"
published: true
---

##  Setting 
In scientific computing, it regularly happens that the computational power
provided by a single laptop or workstation is not enough to perform
the required calculations in an acceptable timeframe. In that
case it is necessary to use more resources as are often provided
in high performance computing environments. In this post, I will
assume that the HPC cluster is run by a scheduler like Grid Engine
or Slurm. But for it to be applicable to only assumption is that
an "array" job can be started from the command line. Even on a
single compute node, the same technique can be used together
with the GNU parallel program on Linux (although there are
other options that are usually more appropriate).


## Implementation

The intent of this post is to show how to transform
a normal python script into a script that easily runs on a
compute cluster. The focus of this post is to do it while
staying as basic as possible, i.e. without using other
packages that would add complexity or make debugging more
difficult. Submission to the cluster is assumed to work through
a submission script as are usually used by systems like Grid Engine.

### Setting

In the following it is assumed that you have a job that
has several long running tasks, where all these tasks are independent
of each other. For example when the same computation has to be performed
for each of a list of files or other parameters. In particular, I
tend to use it for operations on large images or for
doing exploration of meta-parameters in deep learning.

In pseudo-code, we roughly have a setting of the form:

```
...
Functions
...

if __name__ == "__main__":
    Gather information about tasks
        to perform and necessary setup

    for task in task_list:
        perform task
```

### Specify tasks from the command line

The first step is now to ensure that when called from
the command line, we can specify that only certain tasks
have to be performed, where we refer to them by their position
in the list, starting from position 0. This is easily achieved
using the *argparse* module in python.

```python
import argparse
from typing import Optional

def str_to_slice(s: str) -> slice:
    """Convert a string to a slice"""
    if len(s) == 0:
        return slice(0, None)

    s_split = s.split(":")
    s_split = [int(s) for s in s_split]
    if len(s_split) == 1:
        return slice(s_split[0], s_split[0] + 1)
    if len(s_split) in (2, 3):
        return slice(*s_split)
    else:
        raise("Can't convert to a slice {}".format(s))


def get_parsed_args(parser: Optional[argparse.ArgumentParser] = None) -> argparse.Namespace:
    if parser is None:
        parser = argparse.ArgumentParser(
            description="Parsing arguments for tasks"
        )
    parser.add_argument(
        "--tasks",
        dest="tasks",
        type=str_to_slice,
        nargs="?",
        default=slice(0, None),
        help="Specify tasks to perform as int or in slice notation"
    )
    parser.add_argument(
        "--cluster-script",
        dest="cluster_script",
        action="store_true",
        help="Output cluster script but don't perform any task"
    )

    args = parser.parse_args()
    return args
```

Using this function we can now specify a slice on the command
line that should be computed (and by default, the entire
list is used). Our *main* function now looks something like this:

```python
if __name__ == "__main__":
    Gather information about tasks
        to perform and necessary setup

    # subset parameters as requested
    args = get_parsed_args()
    task_list = task_list[args.tasks]

    for task in task_list:
        perform task
```

Now we are already almost there. What is left to do is that we
need a script that we can use with e.g. qsub to submit our job
to the cluster. A typical submission script could look something
like this:
```bash
#!/bin/bash

#$ -N do_something
#$ -pe smp 1
#$ -binding linear:1
#$ -cwd
# Use /bin/sh so that .bashrc does not get read;
#$ -S /bin/sh
#$ -shell yes
#$ -V
# Merge output and error into one file
#$ -j y
#$ -o do_something/$JOB_NAME.o$JOB_ID.$TASK_ID
# Resource parameters
#$ -l m_mem_free=10G
#$ -l h_rt=3600
# Number of tasks for array job
#$ -t 1-14
# Set the queue
#$ -q default.q

umask 007
STEP_SIZE=1
START_TASK=$(((SGE_TASK_ID - 1) * STEP_SIZE))
STOP_TASK=$((START_TASK + STEP_SIZE))
python do_something.py --tasks=${START_TASK}:${STOP_TASK}
```

Of course, in this script we have to enter the number of tasks
that are to be done "by hand" as well as the number of cpus
per task, memory, time requirements as well as other
resources. Therefore, in order to make all
this a bit more convenient, we will parametrize our cluster
submission script and fill it out from inside our
python-script. There are several possible choices of tools
for this. The one I decided to use for its simplicity is
[mustache](https://mustache.github.io/). Once we parametrize our
cluster submission script, this will look like this

{% raw %}
```
#!/bin/bash

#{{#job_name}}$ -N {{{.}}}{{/job_name}}
#$ -pe smp {{{cpus}}}
#$ -binding linear:1
#$ -cwd
# Use /bin/sh so that .bashrc does not get read
#$ -S /bin/sh
#$ -shell yes
#$ -V
# Merge output and error into one file
#$ -j y
#{{#output_dir}}$ -o {{{.}}}/$JOB_NAME.o$JOB_ID.$TASK_ID{{/output_dir}}
{{#wait}}
#$ -sync y
{{/wait}}
# Resource parameters
#$ -l m_mem_free={{{mem_free}}}
#$ -l h_rt={{{h_rt_secs}}}
# Number of tasks for array job
#$ -t 1-{{{num_array_tasks}}}
{{#queue}}
# Set the queue
#$ -q {{{.}}}
{{/queue}}

umask 007
STEP_SIZE={{{step}}}
START_TASK=$(((SGE_TASK_ID - 1) * STEP_SIZE))
STOP_TASK=$((START_TASK + STEP_SIZE))
python {{{script_file}}} --tasks=${START_TASK}:${STOP_TASK}
```
{% endraw %}

and a function to fill it out from inside python could
be something like this

```python
import pystache
from pathlib import Path
from typing import Optional, Any

def write_cluster_script(
    template: Path,
    num_tasks: int,
    script_file: Path,
    output_dir: Optional[Path] = None,
    step: int = 1,
    cpus: int = 1,
    mem_free: str = "4G",
    **kwargs: Any
) -> Path:
    if output_dir is None:
        output_dir = script_file.with_suffix('')

    num_array_tasks = int(math.ceil(float(num_tasks) / step))
    kwargs["num_array_tasks"] = num_array_tasks
    kwargs["step"] = step
    kwargs["script_file"] = str(script_file)
    kwargs["output_dir"] = str(output_dir)
    kwargs['cpus'] = cpus
    kwargs['mem_free'] = mem_free
    if "job_name" not in kwargs:
        kwargs["job_name"] = script_file.stem

    renderer = pystache.Renderer(missing_tags="strict")
    res = renderer.render_path(str(template), kwargs)
    output_dir.mkdir(exist_ok=True, parents=True)
    shell_script = output_dir / "run.sh"
    with (shell_script).open("w") as f:
        f.write(res)

    return shell_script
```
In the parser above you can see that we already added a
switch to the command line interface, so that
we can decide on the command line if we want the submission
script file to be written.

Pulling it all together

```python
if __name__ == "__main__":
    Gather information about tasks
        to perform and necessary setup

    # subset parameters as requeste
    args = get_cmd_args()
    if args.cluster_script:
    	write_cluster_script(... arguments ...)
	    sys.exit(0)
    else:
        task_list = task_list[args.tasks]

    for task in task_list:
        perform task
```

Combined, the supporting functions have less than 100 lines. The change
to the original script is a few added lines and it gives us the ability
from the command line to
- run all tasks
- only specific tasks (also e.g. for debugging)
- create the cluster script that

Overall, we can now easily send our computation to the cluster
while we still stay in complete control of every aspect of the
computation, down to how exactly the computation is distributed.
As an additional bonus, we can run the cluster job as a true
array-job, something that other tools often do not do.

## Tricks

The outline above is dependent on us having a single list 
of all tasks that are to be performed. Sometimes that is 
not directly available. In the case of nested loops, the 
itertools.product function can be useful. 

Another possibly useful tool is the functools.partial function.
This function can be used to delay the execution as a function,
by storing it as a *partial* object and later the computation
can be done by just calling the partial object without any
further addtional arguments or keywords. 

## Other options

Of course there are a whole host of other options that are
available to achieve a similar task, many of them considerably
more sophisticated. On a single node, there is for example
the [multiprocessing](https://docs.python.org/3.8/library/multiprocessing.html) 
module that can be used to distribute
compute tasks to different cores on a single node. For a cluster
setup, options such as [dask](https://dask.org/) could be used and are certainly worth
checking out.
