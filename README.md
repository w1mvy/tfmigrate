# tfmigrate
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub release](https://img.shields.io/github/release/minamijoyo/tfmigrate.svg)](https://github.com/minamijoyo/tfmigrate/releases/latest)
[![GoDoc](https://godoc.org/github.com/minamijoyo/tfmigrate/tfmigrate?status.svg)](https://godoc.org/github.com/minamijoyo/tfmigrate)

A Terraform state migration tool for GitOps.

## Table of content
<!--ts-->
   * [Features](#features)
   * [Why?](#why)
   * [Supported Terraform versions](#supported-terraform-versions)
   * [Getting Started](#getting-started)
   * [Install](#install)
      * [Homebrew](#homebrew)
      * [Download](#download)
      * [Source](#source)
   * [Usage](#usage)
   * [Configurations](#configurations)
      * [Environment variables](#environment-variables)
      * [Configuration file](#configuration-file)
         * [tfmigrate block](#tfmigrate-block)
         * [history block](#history-block)
         * [storage block](#storage-block)
         * [storage block (local)](#storage-block-local)
         * [storage block (s3)](#storage-block-s3)
   * [Migration file](#migration-file)
      * [migration block](#migration-block)
      * [migration block (state)](#migration-block-state)
         * [state mv](#state-mv)
         * [state rm](#state-rm)
         * [state import](#state-import)
      * [migration block (multi_state)](#migration-block-multi_state)
         * [multi_state mv](#multi_state-mv)
   * [License](#license)
<!--te-->

## Features

- GitOps friendly: Write terraform state mv/rm/import commands in HCL, plan and apply it.
- Monorepo style support: Move resources to other tfstates to split and merge easily for refactoring.
- Dry run migration: Simulate state operations with a temporary local tfstate and check to see if terraform plan has no changes after the migration without updating remote tfstate.
- Migration history: Keep track of which migrations have been applied and apply all unapplied migrations in sequence.

You can apply terraform state operations in a declarative way.

In short, write the following migration file and save it as `state_mv.hcl`:

```hcl
migration "state" "test" {
  dir = "dir1"
  actions = [
    "mv aws_security_group.foo aws_security_group.foo2",
    "mv aws_security_group.bar aws_security_group.bar2",
  ]
}
```

Then, apply it:

```
$ tfmigrate apply state_mv.hcl
```

It works as you expect, but it's just a text file, so you can commit it to git.

## Why?

If you have been using Terraform in production for a long time, tfstate manipulations are unavoidable for various reasons. As you know, the terraform state command is your friend, but it's error-prone and not suitable for a GitOps workflow.

In team development, Terraform configurations are generally managed by git and states are shared via remote state storage which is outside of version control. It's a best practice for Terraform.
However, most Terraform refactorings require not only configuration changes, but also state operations such as state mv/rm/import. It's not desirable to change the remote state before merging configuration changes. Your colleagues may be working on something else and your CI/CD pipeline continuously plan and apply their changes automatically. At the same time, you probably want to check to see if terraform plan has no changes after the migration before merging configuration changes.

To fit into the GitOps workflow, the answer is obvious. We should commit all terraform state operations to git.
This brings us to a new paradigm, that is to say, Terraform state operation as Code!

## Supported Terraform versions

The tfmigrate invokes `terraform` command under the hood. This is because we want to support multiple terraform versions in a stable way. Currently supported terraform versions are as follows:

- Terraform v1.0.x
- Terraform v0.15.x
- Terraform v0.14.x
- Terraform v0.13.x
- Terraform v0.12.x

## Getting Started

As you know, terraform state operations are dangerous if you don't understand what you are actually doing. If I were you, I wouldn't use a new tool in production from the start. So, we recommend you to play an example sandbox environment first, which is safe to run terraform state command without any credentials. The sandbox environment mocks the AWS API with `localstack` and doesn't actually create any resources. So you can safely run the `tfmigrate` and `terraform` commands, and easily understand how the tfmigrate works.

Build a sandbox environment with docker-compose and run bash:

```
$ git clone https://github.com/minamijoyo/tfmigrate
$ cd tfmigrate/
$ docker-compose build
$ docker-compose run --rm tfmigrate /bin/bash
```

In the sandbox environment, create and initialize a working directory from test fixtures:

```
# mkdir -p tmp/dir1 && cd tmp/dir1
# terraform init -from-module=../../test-fixtures/backend_s3/
# cat main.tf
```

This example contains two `aws_security_group` resources:

```hcl
resource "aws_security_group" "foo" {
  name = "foo"
}

resource "aws_security_group" "bar" {
  name = "bar"
}
```

Apply it and confirm that the state of resources are stored in the tfstate:

```
# terraform apply -auto-approve
# terraform state list
aws_security_group.bar
aws_security_group.foo
```

Now, let's rename `aws_security_group.foo` to `aws_security_group.baz`:

```
# cat << EOF > main.tf
resource "aws_security_group" "baz" {
  name = "foo"
}

resource "aws_security_group" "bar" {
  name = "bar"
}
EOF
```

At this point, of course, there are differences in the plan:

```
# terraform plan
(snip.)
Plan: 1 to add, 0 to change, 1 to destroy.
```

Now it's time for tfmigrate. Create a migration file:

```
# cat << EOF > tfmigrate_test.hcl
migration "state" "test" {
  actions = [
    "mv aws_security_group.foo aws_security_group.baz",
  ]
}
EOF
```

Run `tfmigrate plan` to check to see if `terraform plan` has no changes after the migration without updating remote tfstate:

```
# tfmigrate plan tfmigrate_test.hcl
(snip.)
YYYY/MM/DD hh:mm:ss [INFO] [migrator] state migrator plan success!
# echo $?
0
```

The plan command computes a new state by applying state migration operations to a temporary state. It will fail if terraform plan detects any diffs with the new state. If you are wondering how the `tfmigrate` command actually works, you can see all `terraform` commands executed by the tfmigrate with log level `DEBUG`:

```
# TFMIGRATE_LOG=DEBUG tfmigrate plan tfmigrate_test.hcl
```

If looks good, apply it:

```
# tfmigrate apply tfmigrate_test.hcl
(snip.)
YYYY/MM/DD hh:mm:ss [INFO] [migrator] state migrator apply success!
# echo $?
0
```

The apply command computes a new state and pushes it to remote state.
It will fail if terraform plan detects any diffs with the new state.

Finally, you can check the latest remote state has no changes with terraform plan:

```
# terraform plan
(snip.)
No changes. Infrastructure is up-to-date.
```

There is no magic. The tfmigrate just did the boring work for you.

## Install

### Homebrew

If you are macOS user:

```
$ brew install minamijoyo/tfmigrate/tfmigrate
```

### Download

Download the latest compiled binaries and put it anywhere in your executable path.

https://github.com/minamijoyo/tfmigrate/releases

### Source

If you have Go 1.15+ development environment:

```
$ git clone https://github.com/minamijoyo/tfmigrate
$ cd tfmigrate/
$ make install
$ tfmigrate --version
```

## Usage

```
$ tfmigrate --help
Usage: tfmigrate [--version] [--help] <command> [<args>]

Available commands are:
    apply    Compute a new state and push it to remote state
    plan     Compute a new state
```

```
$ tfmigrate plan --help
Usage: tfmigrate plan [PATH]

Plan computes a new state by applying state migration operations to a temporary state.
It will fail if terraform plan detects any diffs with the new state.

Arguments:
  PATH               A path of migration file
                     Required in non-history mode. Optional in history-mode.

Options:
  --config           A path to tfmigrate config file
```

```
$ tfmigrate apply --help
Usage: tfmigrate apply [PATH]

Apply computes a new state and pushes it to remote state.
It will fail if terraform plan detects any diffs with the new state.

Arguments
  PATH               A path of migration file
                     Required in non-history mode. Optional in history-mode.

Options:
  --config           A path to tfmigrate config file
```

## Configurations
### Environment variables

You can customize the behavior by setting environment variables.

- `TFMIGRATE_LOG`: A log level. Valid values are `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`. Default to `INFO`.
- `TFMIGRATE_EXEC_PATH`: A string how terraform command is executed. Default to `terraform`. It's intended to inject a wrapper command such as direnv. e.g.) `direnv exec . terraform`.

Some history storage implementations may read additional cloud provider-specific environment variables. For details, refer to a configuration file section for storage block described below.

### Configuration file

You can customize the behavior by setting a configuration file.
The path of configuration file defaults to `.tfmigrate.hcl`. You can change it with command line flag `--config`.

The syntax of configuration file is as follows:

- A configuration file must be written in the HCL2.
- The extension of file must be `.hcl`(for HCL native syntax) or `.json`(for HCL JSON syntax).
- The file must contain exactly one `tfmigrate` block.

An example of configuration file is as follows.

```hcl
tfmigrate {
  migration_dir = "./tfmigrate"
  history {
    storage "s3" {
      bucket = "tfmigrate-test"
      key    = "tfmigrate/history.json"
    }
  }
}
```

#### tfmigrate block

The `tfmigrate` block has the following attributes:

- `migration_dir` (optional): A path to directory where migration files are stored. Default to `.` (current directory).

The `tfmigrate` block has the following blocks:

- `history` (optional): Keep track of which migrations have been applied.

#### history block

The `history` block has the following blocks:

- `storage` (required): A migration history data store

#### storage block

The storage block has one label, which is a type of storage. Valid types are as follows:

- `local`: Save a history file to local filesystem.
- `s3`: Save a history file to AWS S3.

If your cloud provider has not been supported yet, as a workaround, you can use `local` storage and synchronize a history file to your cloud storage with a wrapper script.

#### storage block (local)

The `local` storage has the following attributes:

- `path` (required): A path to a migration history file.

An example of configuration file is as follows.

```hcl
tfmigrate {
  migration_dir = "./tfmigrate"
  history {
    storage "local" {
      path = "tmp/history.json"
    }
  }
}
```

#### storage block (s3)

The `s3` storage has the following attributes:

- `bucket` (required): Name of the bucket.
- `key` (required): Path to the migration history file.
- `region` (optional): AWS region. This can also be sourced from the `AWS_DEFAULT_REGION` and `AWS_REGION` environment variables.
- `access_key` (optional): AWS access key. This can also be sourced from the `AWS_ACCESS_KEY_ID` environment variable, AWS shared credentials file, or AWS shared configuration file.
- `secret_key` (optional): AWS secret key. This can also be sourced from the `AWS_SECRET_ACCESS_KEY` environment variable, AWS shared credentials file, or AWS shared configuration file.
- `profile` (optional): Name of AWS profile in AWS shared credentials file or AWS shared configuration file to use for credentials and/or configuration. This can also be sourced from the `AWS_PROFILE` environment variable.

The following attributes are also available, but they are intended to use with `localstack` for testing.

- `endpoint` (optional): Custom endpoint for the AWS S3 API.
- `skip_credentials_validation` (optional): Skip credentials validation via the STS API.
- `skip_metadata_api_check` (optional): Skip usage of EC2 Metadata API.
- `force_path_style` (optional): Enable path-style S3 URLs (`https://<HOST>/<BUCKET>` instead of `https://<BUCKET>.<HOST>`).

An example of configuration file is as follows.

```hcl
tfmigrate {
  migration_dir = "./tfmigrate"
  history {
    storage "s3" {
      bucket  = "tfmigrate-test"
      key     = "tfmigrate/history.json"
      region  = "ap-northeast-1"
      profile = "dev"
    }
  }
}
```

## Migration file

You can write terraform state operations in HCL. The syntax of migration file is as follows:

- A migration file must be written in the HCL2.
- The extension of file must be `.hcl`(for HCL native syntax) or `.json`(for HCL JSON syntax).

Although the filename can be arbitrary string, note that in history mode unapplied migrations will be applied in alphabetical order by filename. It's possible to use a serial number for a filename (e.g. `123.hcl`), but we recommend you to use a timestamp as a prefix to avoid git conflicts (e.g. `20201114000000_dir1.hcl`)

An example of migration file is as follows.

```hcl
migration "state" "test" {
  dir = "dir1"
  actions = [
    "mv aws_security_group.foo aws_security_group.foo2",
    "mv aws_security_group.bar aws_security_group.bar2",
  ]
}
```

The above example is written in HCL native syntax, but you can also write them in HCL JSON syntax.
This is useful when generating a migration file from other tools.

```json
{
  "migration": {
    "state": {
      "test": {
        "dir": "dir1",
        "actions": [
          "mv aws_security_group.foo aws_security_group.foo2",
          "mv aws_security_group.bar aws_security_group.bar2"
        ]
      }
    }
  }
}
```

### migration block

- The file must contain exactly one `migration` block.
- The first label is the migration type. There are two types of `migration` block, `state` and `multi_state`, and specify one of them.
- The second label is the migration name, which is an arbitrary string.

The file must contain only one block, and multiple blocks are not allowed, because it's hard to re-run the file if partially failed.

### migration block (state)

The `state` migration updates the state in a single directory. It has the following attributes.

- `dir` (optional): A working directory for executing terraform command. Default to `.` (current directory).
- `actions` (required): Actions is a list of state action. An action is a plain text for state operation. Valid formats are the following.
  - `"mv <source> <destination>"`
  - `"rm <addresses>...`
  - `"import <address> <id>"`
- `force` (optional): Apply migrations even if plan show changes

Note that `dir` is relative path to the current working directory where `tfmigrate` command is invoked.

We could define strict block schema for action, but intentionally use a schema-less string to allow us to easily copy terraform state command to action.

Examples of migration block (state) are as follows.

#### state mv

```hcl
migration "state" "test" {
  dir = "dir1"
  actions = [
    "mv aws_security_group.foo aws_security_group.foo2",
    "mv aws_security_group.bar aws_security_group.bar2",
  ]
}
```

#### state rm

```hcl
migration "state" "test" {
  dir = "dir1"
  actions = [
    "rm aws_security_group.baz",
  ]
}
```

#### state import

```hcl
migration "state" "test" {
  dir = "dir1"
  actions = [
    "import aws_security_group.qux qux",
  ]
}
```

### migration block (multi_state)

The `multi_state` migration updates states in two different directories. It is intended for moving resources across states. It has the following attributes.

- `from_dir` (required): A working directory where states of resources move from.
- `from_workspace` (optional): A terraform workspace in the FROM directory. Defaults to "default".
- `to_dir` (required): A working directory where states of resources move to.
- `to_workspace` (optional): A terraform workspace in the TO directory. Defaults to "default".
- `actions` (required): Actions is a list of multi state action. An action is a plain text for state operation. Valid formats are the following.
  - `"mv <source> <destination>"`
- `force` (optional): Apply migrations even if plan show changes

Note that `from_dir` and `to_dir` are relative path to the current working directory where `tfmigrate` command is invoked.

Example of migration block (multi_state) are as follows.

#### multi_state mv

```hcl
migration "multi_state" "mv_dir1_dir2" {
  from_dir = "dir1"
  to_dir   = "dir2"
  actions = [
    "mv aws_security_group.foo aws_security_group.foo2",
    "mv aws_security_group.bar aws_security_group.bar2",
  ]
}
```

## License

MIT
