# Bashistrano

**Bashistrano** is a remote server automation tool heavily inspired by [Capistrano](http://capistranorb.com/). It is written purely in bash without any external dependency.

The main use-case of Bashistrano is to deliver containerized (Docker-based) projects to remote servers. Once it's set up, you can deploy your code as easily as:

```
$ cd project
$ ./bashistrano deploy uat v1.0.2
```

## Getting started

1. Download the `bashistrano` script and commit in your project's source code.
2. Add a `/tmp` entry in your `.gitignore` file
3. Follow the instructions below on how to use and configure Bashistrano.

---

# Contents

* [Usage](#usage)
* [Configuration](#configuration)
  - [Stages](#stages)
  - [SSH Authentication](#ssh-authentication)
  - [Using Docker images](#using-docker-images)
  - [Example configuration](#example-configuration)
* [Phases and Steps](#phases-and-steps)
  - [Step descriptions](#step-descriptions)
* [Hooks](#hooks)
  - [Stage specific hooks](#stage-specific-hooks)
  - [Variables](#variables)
  - [Functions](#functions)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [Author](#author)
* [License](#license)

# Usage

```
$ ./bashistrano

Usage:  ./bashistrano COMMAND

A bash deployment automation tool

Options:
  -h, --help     Display this help message
  -v, --version  Print version information and quit

Commands:
  deploy           Deploys the application end-to-end
  only:prepare     Prepares the environment for deployment
  only:deliver     Delivers from an already prepared environment

Run './bashistrano COMMAND --help' for more information on a command
```

# Configuration

The following parameters can be set in `config.sh` or any stage specific configuration file like `config/uat.sh`.

Stage specific configuration takes precedence over the default defined in `config.sh`.

| Parameter | Description
|-|-|
| `application` | application name |
| `deploy_to` | remote directory for deployments |
| `keep_releases` | number of releases to keep on remote servers |
| `servers` | array of remote servers in `user@host` format |
| `images` | array of dependent Docker images (see more below) |

## Stages

In Bashistrano terminology different deployment environments are called **stages**. The required folder structure is the following for a project with `uat` and `prod` stages.

```
project/
├── config/           <-- stage specific configuration
│   ├── uat.sh
│   └── prod.sh
├── stages/           <-- stage specific code
│   ├── uat/
|   │   └── ...
│   └── prod/
|       └── ...
├── bashistrano       <-- bashistrano script
└── config.sh         <-- main configuration
```

## SSH authentication

It's recommended to use public key authentication on your servers. If it's not possible for some reason, you can supply your SSH password when running Bashinstrano.

```
SSH_PASS=secret ./bashistrano deploy uat v1.0.0
```

When the `SSH_PASS` variable is set Bashistrano is using the [sshpass](https://linux.die.net/man/1/sshpass) utility under the hood.

> *NOTE:* this method is **not recommended** since a simple `ps` command can expose your SSH password to all users on the same host.

## Using Docker images

Bashistrano allows you to specify a list of Docker images which need to be downloaded and packaged together with the deployment code. This way remote servers don't need to have access to any private Docker registries.

The `images` parameter is an array with pairs of Docker tags with the following structure.

```
images=(
  <first image remote> <first image local>
  <second image remote> <second image local>
)
```

The first value is the remote Docker image which need to be downloaded (possibly from a private registry). The second value is the desired tag which will be used on the remote servers.

Bashistrano is matchint local and remote Docker image IDs and performs uploading only if necessary.

## Example configuration

For a typical multi-stage use-case the configuration will look like the following:

Default configuration in `config.sh`

```bash
#!/bin/bash -e

application="myapp"
deploy_to="~/apps/$application/$stage"
keep_releases=5

images=(
  "registry.example.com/myapp:1.0.0" "local/myapp:1.0.0"
  "registry.example.com/dep:1.2.1" "local/dep:1.2.1"
)
```

Stage-specific configuration in `config/uat.sh`

```bash
#!/bin/bash -e

servers=( "user@uat-1.server" "user@uat-2.server" )
```

# Phases and Steps

The deployment process consists of multiple **steps** grouped into two **phases**. The two phases together make up the deploy process.

```
STEP                   PHASE
                                  ──┐
                   ──┐              │
clean_local          │ prepare      │
pull_images          │              │
                   ──┤              │
clean_remote         │              │
push_images          │              │ deploy
push_code            │              │
publish_release      │ deliver      │
cleanup_releases     │              │
log_revision         │              │
                   ──┘              │
                                  ──┘
```

## Step descriptions

| Step | Description |
|-|-|
| `clean_local` | Clean up local workspace |
| `pull_images` | Download specified Docker images and save them locally |
| `clean_remote` | Clean workspace on remote servers |
| `push_images` | Upload locally stored Docker images to remote servers |
| `push_code` | Upload current version of stage specific code |
| `publish_release` | Switch out previous release to the newly uploaded one |
| `cleanup_releases` | Keep only the last N release specified by `keep_releases` parameter |
| `log_revision` | Add new entry to revision log file |

# Hooks

The deployment process can be customized using **hooks**. Hooks are triggered before or after _phases_ and _steps_.

The hooks are executed in the following order.

```
start
├── before:deploy
│   ├── before:prepare
│   │   ├── before:clean_local
│   │   ├── ...
│   │   ├── ...
│   │   └── after:pull_images
│   ├── after:prepare
│   ├── before:deliver
│   │   ├── before:clean_remote
│   │   ├── ...
│   │   ├── ...
│   │   └── after:log_revision
│   └── after:deliver
└── after:deploy
```

You can simply add your custom hook by implementing a method named after the given hook in any of the configuration files.


```bash
#!/bin/bash -e

application="myapp"
keep_releases=5

# ...

after:deliver() {
  run_locally 'echo Yaay'
}
```

> *NOTE:* if you implement a hook function you must specify a body, otherwise the process will fail

## Stage specific hooks

If you want to have custom behaviour only on a specific stage, you have two options.

Let's assume you want to add an `uat` specific `after:deliver` hook.

The hook is already in use in `config.sh`:

```bash
#!/bin/bash -e

after:deliver() {
  run_locally 'echo Hello'
}
```

You can add stage-specific behaviour in `config/uat.sh` using a hook function postfixed with the stage name. It will run **after** the global hook:

```bash
#!/bin/bash -e

after:deliver:uat() {
  run_locally 'echo World!'
}
```

## Variables

In addition to the parameters you've explicitly set in the configuration the following Bash variables are accessible in hooks:

| Variable name | Description |
|-|-|
| `$stage` | name of currently running stage |
| `$version` | VERSION parameter passed on the command line |
| `$release_id` | unique identifier of current run (date+time) |
| `$release_path` | **remote path** for the release being uploaded |
| `$current_path` | **remote path** for the currently active release |
| `$tmp_path` | **remote path** for temporary files (cleaned on `clean_remote` step)
| `$local_tmp_path` | **local path** for temporary files (cleaned on `clean_local` step)
| `$output` | captured standard output of the last invoked function (see [Functions](#functions)) |

## Functions

When writing your hooks you can you can leverage the utility functions listed below.

`run_locally <command>` -- Run command on the local machine. The command being executed is displayed properly on the output.

`run_remotely <command>` -- Run command on **every** remote machine specified in the `servers` array.

`copy_to_remote <local_path> <remote_path>` -- Use `scp` to recursively copy files to **every** remote machine.

`run_with_tunnel <local_port> <server> <remote_addr> <command>` -- Run command locally, but surrounded by opening and closing an SSH tunnel.

# Roadmap

Bashistrano is not yet and (as almost all open-source projects) probably never will be _complete_. The following features can provide significant improvements and are planned to be implemented in the future.

If you consider to lend a helping hand visit the [Contributing](#contributing) section.

**Planned features:**

- parse and handle command line flags
- user input validation
- error handling
- rollback functionality
- check code integrity
- check for new versions
- generate initial folder structure
- generate additional stages

# Contributing

Bashistrano is an open-source project, contributions and feedbacks are welcome as always!

1. Fork it ( http://github.com/team-supercharge/bashistrano/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

# Author

[exoszajzbuk](https://github.com/exoszajzbuk) @ [![Supercharge](https://s18.postimg.org/hmcb95l8p/logo_with_text.png)](http://supercharge.io/)

# License

[MIT](LICENSE.md)

```
Copyright (c) 2018 Supercharge

MIT License

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```
