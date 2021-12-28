How to use bash in docker using configuration files
===================================================

Abstract
--------

When using Docker for data science, a particular problem is to ensure
that the correct environments are loaded at all times, e.g. conda.

In order to ensure that the environment is loaded correctly at all
times, it has to be initalized when starting an interactive shell as
well as when starting a program using a run command.

In the following we will show that this can best be achieved in 2 ways:
a) Always start each interactive shell and each run command using a
login shell. b) For interactive shells define a bashrc-file and set the
environemnt variable $BASH\_ENV to the same file for non-interactive
shells (e.g. run commands).

Type of shells
--------------

Before we see all this in detail, we first review how the bash shell
configuration works. For shells, there are two properties that each
shell has. It is either a *login* or a *non-login* shell as well as an
*interactive* or *non-interactive* shell.

### Login shells

A login shell is typically started when logging into a computer. In it,
certain startup scripts are sourced that can be used to set the initial
values for environment varialbles, e.g. PATH. A new bash shell can be
explicitly turned into a login shell by using the *-l* or *--login*
options.

### Interactive shells

An interactive shell is a shell that has its input, output and error
streams connected to a terminal. This is typically the case when you
start a shell inside another shell or when starting a shell in a docker
container. The typical case of a non-interactive shell is a shell that
is started in order to run a script. The option *-i* can be used to
explicitly turn a shell into an interactive shell.

In addition to these option there are other switches that can be used to
customize the behaviour which startup scripts get run and we will go
over them and their effects later.

Configuration files
-------------------

There are a number of different configuration files that are sourced in
different situations. Here an overview of the ones relevant for bash:

-   **/etc/profile:** This system-wide script is sourced by
    *login*-shells at startup before any other files are sourced
-   **/etc/profile.d:** A system-wide directory from which additional
    scripts are sourced by *login* shells. While not formally listed in
    the GNU manual linked above, most distributions also read all
    scripts in this directory.
-   **~/.bash\_profile, ~/.bash\_login, ~/.profile:** These are scripts
    for individual users that are read by *login* shells. Only the first
    of these scripts that exists and is readable is used. If the option
    *--noprofile* is used, none of these scripts is sourced.
-   **/etc/bashrc or /etc/bash.bashrc:** A system-wide script that is
    sourced by *interactive* shells. CentOS uses */etc/bashrc* whereas
    Debian-based systems use */etc/bash.bashrc*.
-   **~/.bashrc:** This user-specific script is sourced for all
    *interactive* shells. If the option *--norc* is used, this file is
    not being sourced. If the option *--rcfile file* is being used,
    *file* is sourced instead of *~/.bashrc*.
-   **$BASH\_ENV:** If a non-interactive shell is started and the
    environment variable *BASH\_ENV* exists, then the script file
    referenced in *BASH\_ENV* will be sourced.

### Behaviour for sh

When bash is invoked with the name *sh*, then its behviour changes.

-   **login:** This behviour occurs when *sh* is started with the
    *--login* option. It sources */etc/profile* and *~/.profile* in this
    order. The *--noprofile* prevents this (clarify if it prevents
    reading of both files or only one of them).
-   **interactive:** It looks for the environment variable *ENV* and
    sources the file referenced here. The option *--rcfile* has no
    effect.
-   **non-interactive:**: No startup files are being sourced.

### POSIX mode:

When started in POSIX mode, only the file referenced in the variable
*ENV* is sourced. No other files are sourced.

The docker setup
----------------

The rules for when which configuration files are executed for which
shell can be quite challenging to remember - at least for users that
don't use this functionality every day. The setup becomes even more
challenging when used together with Docker, where it is a priori less
clear which type of shell is in use at which point.

In order to make this easier to remember we create a small docker
container that shows which configuration files are run in which order
under different conditions.

In the container, *echo* commands specifying the name of the file being
sourced are either - added at the end of the configuration file -
replace the configuration file with the *echo* command

The reason for these two different conditions is that by default some
configuration files by default source other files and with this setup we
want to highlight the exact connections between files.

### Bash as an interactive shell

We can of course also regularly start bash as an interactive shell in
the docker container usign the *-it* option for the docker-run command.
We also specify to docker to log us in as *userA*.

In this case, */etc/bash.bashrc* and */home/userA/.bashrc* are being
sourced (please note that it doesn't occur as an explicit code-block
here as the interactive part does not work in a Jupyter notebook).

    docker run -it --user userA hhoeflin/shell_test:bash-append /bin/bash

    ## Source /etc/bash.bashrc
    ## Source /home/userA/.bashrc
    ## userA@55411b7d527f:/$ exit

Another option to start an interactive shell is the *-i* option to bash.
This cannot be combined with the bash *-c* option to run a command
passed as a string, but we can use it when executing a script. However
this results in an error

    docker run --user userA hhoeflin/shell_test:bash-append /bin/bash -i /home/userA/script.sh

    ## bash: cannot set terminal process group (-1): Inappropriate ioctl for device
    ## bash: no job control in this shell
    ## Source /etc/bash.bashrc
    ## Source /home/userA/.bashrc
    ## /home/userA

In this case, instead of */etc/bash.bashrc* and */home/userA/.bashrc*,
we can force a specific file to be used with the *--rcfile* option. We
can also ensure that for interactive shells, no configuration script is
loaded with *--norc*.

### The non-interactive bash shell

Instead of the interactive shell, we however usually when running a
container are being dropped into a non-interactive, non-login shell.

    docker run hhoeflin/shell_test:bash-append /bin/bash

As we can see, this sources no files at all. But when we now set the
*BASH\_ENV* variable to */etc/profile*, we see that the script gets
loaded - together with the script file in */etc/profile.d*.

    docker run -e BASH_ENV=/etc/profile hhoeflin/shell_test:bash-append /bin/bash

    ## Source /etc/profile.d/test.sh
    ## Source /etc/profile

Strictly speaking the script in */etc/profile.d* should not have been
loaded - at least we did not explicitly ask for it. The reason is that
all script in */etc/profile.d* get sourced by the default
*/etc/profile*. We confirm this by running the same command, but this
time using the version of the configuration scripts that got replaced
with the echo commands - not appended to.

    docker run -e BASH_ENV=/etc/profile hhoeflin/shell_test:bash-replace /bin/bash

    ## Source /etc/profile

### Bash as a login shell

When running bash, we can ask for it to be a login shell using the *-l*
or *--login* option.

    docker run hhoeflin/shell_test:bash-append /bin/bash --login

    ## Source /etc/profile.d/test.sh
    ## Source /etc/profile

and all the profile-related scripts get sourced. When at the same time
we pass the *--noprofile* option

    docker run hhoeflin/shell_test:bash-append /bin/bash --login --noprofile

it prevents any profile scripts from being loaded.

Now this was all so far for the root user. We can also do this for any
other user

    docker run --user userA hhoeflin/shell_test:bash-append /bin/bash --login

    ## Source /etc/profile.d/test.sh
    ## Source /etc/profile
    ## Source /home/userA/.bash_profile

in which case *~/.bash\_profile* gets loaded, as this is the first
user-specific configuration file. For *userB* and *userC*, we can see
similar results, just with their user-specific config files. For *userD*
that has all 3 profile-configuration files, ony the first,
*~/.bash\_profile* gets loaded.

Summary
-------

We have seen how to use login shell and interactive/non-interactive
shells in a Docker container. If the goal is to have a specific script
run in interactive shells and non-interactive shells executing a
command, then there are basically 2 choices to make this happen.

The first choice is to make interactive shells as well as
non-interactive shells both login shells and put the script that should
be executed into either */etc/profile.d* if it should be run for all
users or into *~/.profile* if it is intended for a specific user. This
however requires to set the *-l* option on all shells that are run.

The second option is to set the script as */etc/bash.bashrc* (or
*/etc/bashrc* on CentOS). In this case, this will automatically be run
for all interactive shells. For non-interactive shells, the *BASH\_ENV*
variable pointing to this file would ensure that it is sourced in this
case as well.

Overall, requiring users to always request a login shell and not include
any *bashrc* files is overall a bit more consistent in my opinion.

References
----------

In order to compile this Dockerfile and writeup I used various sources
on the internet.
- A good introduction to the subject is [GNU - Bash Startup Files](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html).
- Another very nice post is [Bash interactive, login shell types](https://transang.me/bash-interactive-login-shells/)
- [Bash cheat sheet](https://www.pcwdld.com/bash-cheat-sheet) (thanks to Marc Wilson for the link).


Updates
-------
- Added bash cheat-sheet on Dec 28th 2021 on suggestion of Marc Wilson.
