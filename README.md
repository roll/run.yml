# run.yml

[![Version](https://img.shields.io/badge/Version-v0.2-orange.svg)](https://github.com/goodread/goodread#changelog)

Task runner for the 21st century. YML config. Shell commands. Sync/async tasks. And much more.

## Motivation

A huge amount of teams uses as a task runner something that's not really intended for it like `make` or `npm`. Or something really heavy weighted for very simple tasks. It partially works but we definitely can do better.

## Features

- simple YML-config like `travis.yml`
- everything are just plain shell commands
- sequential/parallel/multiplex task execution
- environment variables setting with dotenv support
- named/optional subtasks with  cherry-picking support
- arguments goes to the target command with no special syntax
- support for autocompletion, abbreviations and task descriptions
- powerful help system providing execution plans and more
- implementations for Python, JavaScript, Ruby and PHP

## Example

> run.yml

```yml
# Vars

# It's just an environment variable
# It will be available for all tasks
VERSION: head -n 1 run/VERSION
RUNVARS: echo .env  # dotenv


# Tasks

# Here we write task descriptions
# And this task is old good sequential task
pandoc:
  - echo $VERSION
  - pandoc --version
  - pandoc -f markdown_github -t rst -o README.rst README.md
  - mv README.rst README.md


# This task will be executed in parallel (fast)
# But output will preserve commands order (readable)
(test):
 lint!: eslint --no-ignore  # quiet
 unit: NODE_ENV=testing mocha $RUNARGS


# This task will be executed in multiplex mode
# Remember a situation when you open a lot of terminals
((dev)):
  - backend:
    - lint: pylama --watch
    - unit: pytest --cov goodtablesio --cov-report --watch
  - frontend:
    - lint: eslint --watch
    - unit: NODE_ENV=testing karma start --watch
    - /e2e: nightwatch start  # optional
```

> CLI

```shell
$ run # list tasks
$ run test # run task
$ run test ? # print task help
$ run test unit # run subtask
$ run test unit --debug # pass arguments to the target command
$ run test --debug # note $RUNARGS to specify command for arguments
$ run dev +e2e # run task with optional subtask
$ run dev -lint # run task without matching subtasks
$ run dev =unit # run task with only matching subtask
$ run VERSION # print variable value
$ run t[TAB] # use autocompletion
$ run t # use task abbreviations
$ run tu # even for subtasks!
```

## Documentation

### Installation

The packages use semantic versioning. It means that major versions could include breaking changes. It’s highly recommended to specify a package version range in your dependencies.

```bash
$ pip install run.yml # Python
$ npm install run.yml # JavaScript
$ gem install run.yml # Ruby
$ composer require runyml/run.yml # PHP
```

### Configuration

The task runner uses simple YML-based format for the task configuration file. It makes possible to use comments and any other YML syntax. This file must be named `run.yml`:

```yml
VARIABLE: command

# Main task
task:
  - command1
  - command2
```

Structurally it's a dictionary of variables and tasks with shell commands on the right side. Tasks could be composite containing list of subtasks. We use a `run` CLI command in current working directory to run it:

```shell
$ run # list tasks
$ run task # run task
$ run VARIABLE # print variable
````

The `run.yml` file could contain second YML document with general `run` options. This document should be a dictionary of keys and values:

```yml
task:
  - command1
  - command2
---
streamline: true
```

### Variable

A variable allows to save a command output into an environment variable. This environment variable could be used in any tasks below:

```yml
VARIABLE: command

# Use variable in tasks
task: echo $VARIABLE
```

It's important to understand that unlike `make` after variable's command execution it becomes just a normal environment variable that could be used on a general basis.

```shell
$ run VARIABLE # print variable
```

When we "run" a variable it always happens in a quiet mode. It means there is no `run`'s log messages and other complimentary information.

### Task

The main actor is the whole system - a task. It could be associated with a single command or uses multiple commands:

```yml
# single
task1: command

# composite
task2:
  - command1
  - command2
```

Behaviour is exactly what you could expect. This format is used by many of CI-services like Travis or CircleCI. Probably the most important thing about `run.yml` that there is no any magic in right-side command interpretation. It always used as it is:

```shell
$ run task1 # run single task
$ run task2 # run composite task
```

In case of a failure it will exit with a return code `1` otherwise with a return code `0`. It's tru for any `run` CLI call.

### Parallel task

Things are starting to be more interesting when our composite task contains independent commands. So we could run it in parallel using single parenthesis:

```yml
(task):
  - command1
  - command2
```

In this case command will be runned in parallel but output preserves commands order.

```shell
$ run task # run commands in parallel
command1 output
command2 output
```

This mode gets the best parts from both (sync and async) worlds - fast execution with readable output.

### Multiplex task

Sometimes we want commands to be runned in parallel and we want to get real-time output from all of them. Consider watching tasks for a linter and a test runner. For this case we use double parenthesis and multiplex task:

```yml
((task):
  - command1
  - command2
```

Output will be similar to what `docker-compose`-like software uses. But instead of different containers output here we have output from different long-running commands:

```shell
$ run task # run commands in parallel with real-time output
task 1: command1 output
task 1: command1 output
task 2: command2 output
task 1: command1 output
...
```

### Named subtasks

At the moment we have already worked with subtasks. Every composite task contains subtasks but if it's not named we just consider it as a plain command. Let's name it:

```yml
task:
  - subtask1: command1
  - subtask2: command2
```

Here we touch on of the most important `run.yml` feature - we could create a composite tasks without losing an ability to run its subtasks:

```shell
$ run task # run command1 and command2
$ run task subtask1 # run command1
$ run task subtask2 # run command2
```

### Optional subtask

But what if some subtask logically belongs to some task but we want to run it be default as a part of the containing task? We could use optional subtasks:

```yml
task:
  - subtask1: command1
  - /subtask2: command2
```

By default `run` will skip all optional subtasks for the task:

```shell
$ run task # run command1
$ run task subtask1 # run command1
$ run task subtask2 # run command2
$ run task +subtask2 # run command1 and command2
```

Here we've used one of three task calling modifiers. It's a plus sign to say that we want to include optional subtask to the task execution plan.

### Quiet task/subtask

By default for tasks `run` will print some high-level logging information about launching commands etc. Consider a task:

```yml
task:
  - command
```

Running it we could see:

```shell
$ run task
[run] Prepared "RUNARGS="
[run] Launched "command"
command output
[run] Finished in 0.272 seconds
```

To prevent the logging we use an exclamation point at the task name end (similiar to `@` modifier in `make`):

```yml
task!:
  - command
```

In this case `run` will not emit any complimentary information:

```shell
$ run task
command output
```

Note that variables are always quiet.

### Cherry-picking subtasks

In the `optional subtask` section we have just used a plus sign to enable optional subtask execution. There are two other CLI modifiers to filter and pick subtasks. It's a equal sign and a minus sign. Let's see on example:

```yml
task:
  - subtask1:
    - nested1: command1
    - nested2: command2
  - subtask2:
    - nested1: command3
    - nested2: command4
    - /nested3: command5
```

Here we use nested subtasks which is totally OK for the `run` task runner. Let's use CLI modifiers:

```shell
$ run task # run command1-4
$ run task +nested5 # run command1-5
$ run task =nested1 # run command1 and command3
$ run task -nested1 # run command2 and command4
```

This syntax is pretty self-explained but provides a very powerful capabilities. Think about it as a kind of small task query language.

### Passing arguments

Passing arguments to a target command is one of the main pain points in probably all existent task runners. It could be something like this `tox -- --debug`. Here `run` uses a completely different approach. Consider a task:

```yml
task:
  - subtask1: command1
  - subtask2: command2 $RUNARGS file
```

First of all `run` passes all arguments directly to the task/subtask without any special CLI syntax:

```shell
$ run task subtask1 --debug # run command1 --debug
$ run task subtask2 --debug # run command2 --debug file
```

Also we use here a special environment variable `RUNARGS` containing passed arguments. It's also very useful when we run a composite task:

```shell
$ run task --debug # run command1 and command2 --debug file
```

By default `$RUNARGS` is added at the end of every command not having it. So there is no magic or something like this - it's just an environment variable.

### Using dotenv file

A technique named `dotenv` is a nifty way to load a list of variables (often secret) from a file into the current environment. There is a special variable called `RUNVARS` to support `dotenv` file:

```yml
RUNVARS: echo .env
```

Here we tell to `run` that variables should be read from the `.env` file. Because as usual the right side is just a normal shell command we could set it dynamically:

```yml
RUNVARS: echo .env_${ENV_TYPE}
```

Read more about `dotenv` for example here - https://github.com/theskumar/python-dotenv.

### Task descriptions

We could add task descriptions using regular YML-syntax for comments. We need to place it straight above the describing task:

```yml
# Main task
task1: command1

# Additional task
# Here we go multiline:
# - and could use something
# - like this list
task2:
  - command2
  - command3
```

In the next section we learn how to show this description.

### Using help system

It's easy guess that there should be a help system. And of course `run` provides this capability. Let's see on an example again:

```yml
VARIABLE: command

# It is our main task!
(task):
  - subtask1: command1
  - subtask2: command2
  - /subtask3: command3
```

First we just want to get information about all our tasks:

```shell
$ run
run

---

Description

General run description.

Vars

run VARIABLE

Tasks

run (selected)
run task
run task subtask1
run task subtask2
run task subtask3 (optional)
```

Now we add a task name and a question mark:

```shell
$ run task ?
run task

---

Description

It is our main task!

Tasks

run task (selected)
run task subtask1
run task subtask2
run task subtask3 (optional)

Execution plan

VARIABLE="command"
[PARALLEL]
  command1 $RUNARGS
  command2
```

And for a subtask:

```shell
$ run task subtask1 ?
run task subtask1

---

Description

It is our main task!

Tasks

run task
run task subtask1 (selected)
run task subtask2
run task subtask3 (optional)

Execution plan

VARIABLE="command"
command1 $RUNARGS
```

One of the most important things here is an execution plan allowing us to understand what exact commands will be executed.

### Abbreviations

That's very nice that software like `npm` allow to run tasks by its first letters. It could be something like `npm t` standing for `npm test`. It's supported in `run` also:

```yml

(test):
  - lint: command1
  - unit: command2
```

So we could use a short version of the command. But even more - we could use it for subtasks (first match wins):

```bash
$ run t # run command1 and command2
$ run tl # run command1
$ run tu # run command2
```

### Autocompletion

For an autocompletion support just add this code to your `~/.bashrc` file and (run `source ~/.bashrc` to activate without terminal re-opening):

```bash
# Run autocompletion
_run()
{
    local list
    local cur

    list=$(run "${COMP_WORDS[@]:1}" --run-complete)
    cur=${COMP_WORDS[COMP_CWORD]}

    COMPREPLY=($(compgen -W '${list}' -- $cur))
}

complete -F _run run
complete -F _run r # if you use an alias "r=run"
```

Now you could be able to use `Tab` to complete run tasks and subtasks:

```shell
$ run t[TAB] # use task autocompletion
$ run task s[TAB] # use subtask autocompletion
```

### General options and arguments

As mentioned in the `run.yml` section this file could contain general `run` options. It could be related to an execution process, optimization or other `run` details:

```yml
# Tasks

task:
  - command1
  - command2
---

# Options

streamline: true
```

Also for CLI calls general arguments could be passed using `--run-<argument>` prefix:

```shell
$ run --run-path configs/run.yml
```

For now there are no offically supported options and CLI arguments. But concrete implementations could provide it as a provisional API.

## Contributing

This project uses consolidated issue tracker and development model. Please use this repository for everything except test runners code.

### Python

Requirements:
- installed `virtualenv` - https://virtualenv.pypa.io/en/stable/installation/

```bash
git clone git@github.com:runyml/run.yml-py.git
cd run.yml-py
virtualenv .python -ppython3.5
source .python/bin/activate
make install
make test
```

### JavaScript

Requirements:
- installed `nvm` - https://github.com/creationix/nvm#installation

```bash
git clone git@github.com:runyml/run.yml-js.git
cd run.yml-js
nvm install 8
nvm use 8
npm install
npm test
```

### Ruby

Requirements:
- installed `rvm` - https://rvm.io/rvm/install

```bash
git clone git@github.com:runyml/run.yml-rb.git
cd run.yml-rb
rvm install 2.4
rvm use 2.4
gem install bundler
./bin/setup
rake spec
```

### PHP

Requirements:
- installed `phpbrew` - http://phpbrew.github.io/phpbrew/

```bash
git clone git@github.com:runyml/run.yml-php.git
cd run.yml-php
phpbrew install 7.1.0
phpbrew use 7.1.0
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '544e09ee996cdf60ece3804abc52599c22b1f40f4323403c44d44fdfdd586475ca9813a858088ffbc1f233e9b180f061') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php --filename=composer
php -r "unlink('composer-setup.php');"
php composer install
php composer test
```

## Changelog

Here described only breaking and the most important changes. The full changelog and documentation for all released versions could be found in nicely formatted [commit history](https://github.com/runyml/run.yml/commits/master).

### v0.2

Proof of concept version.
