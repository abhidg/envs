# envs -- `.env` sharing within a team

`envs` implements a common use case that is faced by small teams -- how to share
`.env` files, without relying on cloud based solutions (often not free) such as
Hashicorp Vault and AWS Secrets Manager.

Tho approach taken by `envs` is inspired by [`pass`](https://passwordstore.org)
and stores `.env` files in a Git repository. Encryption is handled by `age`
which handily allows encryption to a set of SSH keys. As most code forges have
a easy way of obtaining user keys, this makes it convenient for team members to
use the username as a recipient.

The only dependencies of `envs` are `bash`, `git`, `curl` (usually installed on
most developer systems) and `age`, which can be installed through your package
manager.

> [!WARNING]
> This is software in development and may contain bugs. Currently only files
> in the **root** of a Git repo can be encrypted.

## Installation

Copy the `envs` script somewhere on your PATH, e.g. `~/bin` or `~/.local/bin`.

## Usage

### Initial setup

The initial setup only needs to be performed once. To initialise from a remote
shared repository for env files:
```shell
envs init git@your-server:envs.git
```

You will also need to set your primary decryption key. If the data was encrypted
against your SSH public key from a forge, it will need to be the corresponding
private key (e.g. `~/.ssh/id_ed25519` for Ed25519 keys). This needs to be done
once:
```shell
envs privkey ~/.ssh/id_ed25519
```

### Use `envs` in a repo

Running `envs` pulls all environment variables for that repo into the current
folder named the same as the repo:

```shell
> pwd
/home/abhidg/git/top-secret-project
> envs
Updating 5524541..cff1e7a
Fast-forward
 top-secret-project.env.local.age           |   BIN
 1 files changed
 create mode 100644 top-secret-project.env.local.age
updated top-secret-project/.env.local
```

If the file is already present and has a earlier modified time than the
fetched file, the user is prompted to replace, otherwise warning mentioned the
`.envs.local` is newer than repo. In all cases, no output is emitted if there are
no differences.

If there is no `envs` configuration in a repo, `envs` will prompt you to setup
an initial list of users or keys who will have access to env files. To configure
`envs` in a repo to allow certain keys or forge users:

```shell
envs forge gitlab.com  # set if not using GitHub

# To allow these users
envs addkeys user1 user2 key1

# This will create (or append to) a `.envs_keys.txt` file in the current directory
# It is recommended to check this into version control
git add .envs_keys.txt
git commit .envs_keys.txt -m 'dev: add envs keys file'
git push
```

### Adding `.env` files

Once the repository setup is complete, adding a new env file:

```shell
envs commit .env.local <optional message>
```

This will create a corresponding encrypted file in the Git repo
and push it to the remote.

### Rolling back changes

Changed that password on the database by accident, or need to look at a log of
how a particular file has changed? No problem, `envs` uses the `git log` and
`git checkout` under the hood to support such operations:

```shell
envs log <file>
```
shows the log of env files for the current repo, or for a particular file

### Setting repo

Optionally a repo can be set -- this is the initial part of the encrypted
filename that is used to distinguish env files in different repositories. For
example a `.env.local` in `top-secret-project` would get a encrypted filename
of `top-secret-project.env.local.age`. If that name was taken already in the
Git repo, you have to use a prefix to disambiguate, which is indicated by a special
comment in the first line of `.envs_keys.txt`

```shell
# repo=midlevel-secret-project
key1
# username
key2
```

## Alternatives

[sops](https://github.com/getsops/sops) -- this is a very good tool that can
handle more complex use-cases such as leaving the keys in key=value pairs
unencrypted, as well as supporting multiple encryption mechanisms including GCP
and KMS. In contrast, `envs` is extremely simple (a single shell script) and has
a easy to use interface, relying on code forges to provide a set of public keys for
each user.

## Supported code forges

- GitHub: `envs forge github.com`
- GitLab (including self hosted): `envs forge gitlab.com`
- Codeberg: `envs forge codeberg.org`
