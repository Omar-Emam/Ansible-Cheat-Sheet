# Ansible Cheat Sheet

Last checked: November 3, 2026

Every CLI option here was read from the tools themselves, using `ansible` 14.5.0 with `ansible-core` 2.21.6. The core CLI help in `ansible-core` 2.22.0 is the same, apart from the wider default for `ansible-config validate`.

## Contents

- [Install and quick start](#install-and-quick-start)
- [CLI tools](#cli-tools)
- [Inventory](#inventory)
- [Playbooks and tasks](#playbooks-and-tasks)
- [Variables](#variables)
- [Templates, filters, tests and lookups](#templates-filters-tests-and-lookups)
- [Modules by category](#modules-by-category)
- [Roles and collections](#roles-and-collections)
- [Vault](#vault)
- [Plugins](#plugins)
- [Configuration](#configuration)
- [Debugging, testing and best practices](#debugging-testing-and-best-practices)
- [Sources](#sources)

## Install and quick start

`ansible` is the full package with community collections. `ansible-core` is the engine, the CLI tools and the `ansible.builtin` content only.

| Package | Version checked | Python on control node | Contains |
| --- | --- | --- | --- |
| `ansible` | 14.5.0 | 3.12 or later | `ansible-core` plus a curated set of collections |
| `ansible-core` | 2.21.6 | 3.12 or later | Engine, CLI tools, `ansible.builtin` |
| `ansible-core` | 2.22.0 | 3.13 or later | Newest core release at the time of checking |

```bash
pipx install --include-deps ansible      # Full package, isolated (recommended)
pip install --user ansible               # Full package with pip
pip install ansible-core                 # Engine only
pipx inject ansible ansible-lint         # Add a tool to the pipx environment
pip install --upgrade ansible            # Upgrade
ansible --version                        # Show versions, config file and paths
```

```bash
ansible all -i 'host1,host2,' -m ping             # Test reachability without an inventory file
ansible all -i inventory.ini -m ansible.builtin.setup   # Gather all facts
ansible-playbook -i inventory.ini site.yml --check --diff   # Dry run with diffs
ansible-playbook -i inventory.ini site.yml        # Apply the playbook
```

The trailing comma in `-i 'host1,host2,'` tells Ansible the value is a host list, not a file.

## CLI tools

The core package ships nine user tools plus `ansible-test`. The rest are separate packages from the Ansible developer tooling.

| Tool | Package | Purpose |
| --- | --- | --- |
| `ansible` | core | Run one module against hosts (ad hoc) |
| `ansible-playbook` | core | Run playbooks |
| `ansible-console` | core | Interactive ad hoc shell |
| `ansible-pull` | core | Pull a repo and run it locally |
| `ansible-inventory` | core | Show or export inventory |
| `ansible-galaxy` | core | Manage roles and collections |
| `ansible-vault` | core | Encrypt and decrypt data |
| `ansible-doc` | core | Read plugin documentation |
| `ansible-config` | core | View and validate configuration |
| `ansible-test` | core | Test collections |
| `ansible-lint` | ansible-lint | Lint playbooks and roles |
| `ansible-navigator` | ansible-navigator | Text UI and execution environment runner |
| `ansible-builder` | ansible-builder | Build execution environment images |
| `ansible-runner` | ansible-runner | Run Ansible from code with stable artifacts |
| `molecule` | molecule | Test roles and collections |

### Common options

These options are shared by `ansible`, `ansible-playbook` and `ansible-console`. `ansible-pull` takes most of them too.

| Option | Description |
| --- | --- |
| `-h`, `--help` | Show help |
| `--version` | Show versions, config file and module paths |
| `-v`, `-vv`, `-vvv`, `-vvvv` | Raise verbosity; start at `-vvv`, use `-vvvv` for connection debugging |
| `-i`, `--inventory`, `--inventory-file` | Inventory path or comma-separated hosts; repeatable |
| `-l`, `--limit` | Further limit hosts to a pattern |
| `--list-hosts` | List matching hosts and do nothing else |
| `-e`, `--extra-vars` | Set variables as `key=value`, YAML, JSON or `@file`; repeatable |
| `-f`, `--forks` | Parallel processes (default 5) |
| `-M`, `--module-path` | Prepend module library paths; repeatable |
| `-C`, `--check` | Predict changes without making them |
| `-D`, `--diff` | Show file and template differences |
| `--flush-cache` | Clear the fact cache for every inventory host |
| `--vault-id` | Vault identity to use; repeatable |
| `-J`, `--ask-vault-password`, `--ask-vault-pass` | Prompt for the vault password |
| `--vault-password-file`, `--vault-pass-file` | Read the vault password from a file or script |

**Connection options**

| Option | Description |
| --- | --- |
| `-u`, `--user` | Remote user |
| `-c`, `--connection` | Connection plugin (default `ssh`) |
| `-T`, `--timeout` | Connection timeout in seconds |
| `-k`, `--ask-pass` | Prompt for the connection password |
| `--connection-password-file`, `--conn-pass-file` | Read the connection password from a file |
| `--private-key`, `--key-file` | SSH private key file |
| `--ssh-common-args` | Extra arguments for sftp, scp and ssh, such as `ProxyCommand` |
| `--ssh-extra-args` | Extra arguments for ssh only |
| `--sftp-extra-args` | Extra arguments for sftp only |
| `--scp-extra-args` | Extra arguments for scp only |

**Privilege escalation options**

| Option | Description |
| --- | --- |
| `-b`, `--become` | Run with privilege escalation |
| `--become-method` | Escalation method (default `sudo`) |
| `--become-user` | Target user (default `root`) |
| `-K`, `--ask-become-pass` | Prompt for the become password |
| `--become-password-file`, `--become-pass-file` | Read the become password from a file |

### ansible

```bash
ansible <pattern> [options]
ansible web -m ansible.builtin.ping                         # Check connectivity
ansible web -a 'uptime'                                     # Run a command (default module)
ansible web -b -m ansible.builtin.apt -a 'name=nginx state=present'   # Install a package
ansible web -m ansible.builtin.setup -a 'filter=ansible_distribution*'   # Filter facts
ansible web -m ansible.builtin.copy -a 'src=a dest=/tmp/a'  # Copy a file
ansible web -B 3600 -P 0 -a '/usr/bin/long_job'             # Fire and forget
ansible all --list-hosts                                    # Show matched hosts
```

| Option | Description |
| --- | --- |
| `-m`, `--module-name` | Module to run (default `command`) |
| `-a`, `--args` | Module arguments in `k=v` format, or JSON |
| `-B`, `--background` | Run asynchronously, fail after this many seconds |
| `-P`, `--poll` | Poll interval with `-B` (default 15; 0 means do not wait) |
| `-o`, `--one-line` | Condense output |
| `-t`, `--tree` | Log output to this directory |
| `--task-timeout` | Task time limit in seconds |
| `--playbook-dir` | Substitute playbook directory for relative paths |

### ansible-playbook

```bash
ansible-playbook [options] playbook.yml [playbook2.yml ...]
ansible-playbook site.yml --syntax-check                 # Validate syntax
ansible-playbook site.yml --list-tasks --list-tags       # Preview tasks and tags
ansible-playbook site.yml -l web01 -t nginx              # One host, one tag
ansible-playbook site.yml --start-at-task 'Install app'  # Resume at a task
ansible-playbook site.yml -e 'env=prod' -e @vars.yml     # Pass variables
ansible-playbook site.yml --step                         # Confirm each task
```

| Option | Description |
| --- | --- |
| `-t`, `--tags` | Run only these tags; repeatable |
| `--skip-tags` | Skip these tags; repeatable |
| `--list-tags` | List available tags |
| `--list-tasks` | List tasks that would run |
| `--syntax-check` | Check syntax without running |
| `--step` | Confirm each task before it runs |
| `--start-at-task` | Start at the task with this name |
| `--force-handlers` | Run handlers even if a task fails |

### ansible-console

```bash
ansible-console web -b        # Open a shell against group web, with become
```

It takes the common options, plus `--step`, `--task-timeout` and `--playbook-dir`. Type any module name to run it, or a shell command to run it through `shell`.

| Console command | Description |
| --- | --- |
| `cd <pattern>` | Change the host pattern |
| `list` | List the current hosts; `list groups` lists groups |
| `become`, `become_user`, `become_method` | Set privilege escalation |
| `remote_user` | Set the connection user |
| `check`, `diff` | Toggle check or diff mode |
| `forks`, `serial` | Set parallelism |
| `timeout` | Set the task timeout |
| `verbosity` | Set the verbosity level |
| `collections` | Set the collection search list |
| `shell` | Run a shell command explicitly |
| `help <module>` | Show help for a module |
| `exit` | Leave the console |

### ansible-pull

```bash
ansible-pull -U https://github.com/org/repo.git local.yml               # Pull and run
ansible-pull -U <url> -C main -d /opt/ansible -o -s 60 local.yml        # Run from cron, only on change
```

| Option | Description |
| --- | --- |
| `-U`, `--url` | Repository URL |
| `-C`, `--checkout` | Branch, tag or commit |
| `-d`, `--directory` | Checkout directory |
| `-m`, `--module-name` | Repository module: `git`, `subversion`, `hg` or `bzr` |
| `-o`, `--only-if-changed` | Run only if the repository changed |
| `-s`, `--sleep` | Sleep a random time up to this many seconds |
| `-f`, `--force` | Run even if the repository could not be updated |
| `--full` | Full clone instead of shallow |
| `--clean` | Discard local modifications in the checkout |
| `--purge` | Delete the checkout after the run |
| `--accept-host-key` | Add the repository host key |
| `--verify-commit` | Verify the commit GPG signature |
| `--track-subs` | Submodules track the latest changes |
| `-t`, `--tags`, `--skip-tags` | Filter by tags |
| `--check`, `--diff` | Dry run and show differences |

### ansible-inventory

```bash
ansible-inventory -i inv --list -y              # Full inventory as YAML
ansible-inventory -i inv --graph --vars         # Tree with variables
ansible-inventory -i inv --host web01           # Variables for one host
ansible-inventory -i inv --graph web            # Tree of one group
ansible-inventory -i inv --list --export --output out.json   # Export to a file
```

| Option | Description |
| --- | --- |
| `--list` | Output all hosts |
| `--host` | Output one host |
| `--graph` | Output a tree; accepts a group name |
| `-y`, `--yaml` | YAML output |
| `--toml` | TOML output |
| `--vars` | Add variables to `--graph` |
| `--export` | Optimize `--list` output for export |
| `--output` | Write `--list` output to a file |
| `-l`, `--limit` | Limit hosts to a pattern |
| `--playbook-dir` | Substitute playbook directory |

### ansible-galaxy

```bash
ansible-galaxy collection install community.general            # Install a collection
ansible-galaxy collection install community.general:==10.0.0   # Exact version
ansible-galaxy collection install -r requirements.yml          # Install from a file
ansible-galaxy role install -r requirements.yml -p roles/      # Install roles from a file
ansible-galaxy role init my_role                               # Scaffold a role
ansible-galaxy collection init my_ns.my_col                    # Scaffold a collection
ansible-galaxy collection list                                 # List collections
```

**Options on all `ansible-galaxy` subcommands**

| Option | Description |
| --- | --- |
| `-s`, `--server` | Galaxy API server URL |
| `--token`, `--api-key` | API token |
| `-c`, `--ignore-certs` | Ignore SSL certificate errors |
| `--timeout` | Server timeout |

**Collection subcommands**

| Subcommand | Description | Key options |
| --- | --- | --- |
| `collection install` | Install from Galaxy, file or URL | `-r`, `-p`, `-f`, `--force-with-deps`, `-U`, `-n`, `-i`, `--pre`, `--offline`, `--no-cache`, `--clear-response-cache` |
| `collection download` | Download tarballs for offline install | `-r`, `-p`, `-n`, `--pre`, `--no-cache`, `--clear-response-cache` |
| `collection init` | Create the base structure | `-f`, `--init-path`, `--collection-skeleton`, `-e` |
| `collection build` | Build a tarball | `-f`, `--output-path` |
| `collection publish` | Publish a tarball | `--no-wait`, `--import-timeout` |
| `collection list` | List installed collections | `-p`, `--format` with `human`, `yaml` or `json` |
| `collection verify` | Compare installed checksums with the server | `-r`, `-p`, `-i`, `--offline` |

`collection install` and `collection verify` also take the signature options: `--keyring`, `--signature`, `--required-valid-signature-count`, `--ignore-signature-status-code` and `--ignore-signature-status-codes`. `collection install` adds `--disable-gpg-verify`.

**Role subcommands**

| Subcommand | Description | Key options |
| --- | --- | --- |
| `role install` | Install from Galaxy, file or URL | `-r`, `-p`, `-f`, `--force-with-deps`, `-n`, `-i`, `-g` |
| `role init` | Create the base structure | `-f`, `--offline`, `--init-path`, `--role-skeleton`, `--type`, `-e` |
| `role list` | List installed roles | `-p` |
| `role remove` | Delete local roles | `-p` |
| `role info` | Show role details | `-p`, `--offline` |
| `role search` | Search Galaxy | `--platforms`, `--galaxy-tags`, `--author` |
| `role import` | Import a role into a Galaxy server | `--no-wait`, `--branch`, `--role-name`, `--status` |
| `role setup` | Manage the Galaxy and source integration | `-p`, `--remove`, `--list` |
| `role delete` | Remove a role from Galaxy, not from GitHub | None |

### ansible-vault

The Vault section below shows full usage.

| Subcommand | Description | Extra options |
| --- | --- | --- |
| `create` | Create an encrypted file | `--encrypt-vault-id`, `--skip-tty-check` |
| `edit` | Edit an encrypted file | `--encrypt-vault-id` |
| `view` | View an encrypted file | None |
| `encrypt` | Encrypt a file | `--output`, `--encrypt-vault-id` |
| `decrypt` | Decrypt a file or string | `--output` |
| `encrypt_string` | Encrypt a string | `-p`, `--show-input`, `-n`, `--stdin-name`, `--output`, `--encrypt-vault-id` |
| `rekey` | Change the password | `--encrypt-vault-id`, `--new-vault-password-file`, `--new-vault-id` |

Every subcommand accepts `--vault-id`, `-J` and `--vault-password-file`.

### ansible-doc

```bash
ansible-doc ansible.builtin.copy                # Module documentation
ansible-doc -s ansible.builtin.copy             # Playbook snippet
ansible-doc -l                                  # List modules
ansible-doc -l community.general                # List one collection
ansible-doc -t lookup -l                        # List lookup plugins
ansible-doc -t keyword when                     # Explain a playbook keyword
ansible-doc -t role -l -r roles/                # List roles
```

| Option | Description |
| --- | --- |
| `-t`, `--type` | Plugin type: `become`, `cache`, `callback`, `cliconf`, `connection`, `httpapi`, `inventory`, `lookup`, `netconf`, `shell`, `vars`, `module`, `strategy`, `test`, `filter`, `role`, `keyword` |
| `-l`, `--list` | List plugins; accepts a namespace or collection |
| `-F`, `--list_files` | List plugins with their source files |
| `-s`, `--snippet` | Show a playbook snippet |
| `-j`, `--json` | JSON output |
| `-r`, `--roles-path` | Roles directory; repeatable |
| `-e`, `--entry-point` | Role entry point |
| `-M`, `--module-path` | Prepend module paths |
| `--playbook-dir` | Substitute playbook directory |

### ansible-config

```bash
ansible-config view                       # Show the active ansible.cfg
ansible-config dump --only-changed        # Show settings that differ from defaults
ansible-config list                       # List every setting with docs
ansible-config init --disabled > ansible.cfg   # Generate a commented config
ansible-config validate                   # Check the config file and environment
```

| Subcommand | Description | Options |
| --- | --- | --- |
| `list` | Print all settings | `-c`, `-t`, `-f` with `json` or `yaml` |
| `dump` | Print the effective configuration | `-c`, `-t`, `--only-changed`, `--changed-only`, `-f` with `json`, `yaml` or `display` |
| `view` | Print the config file | `-c`, `-t` |
| `init` | Create an initial config | `-c`, `-t`, `-f` with `ini`, `env` or `vars`, `--disabled` |
| `validate` | Check the config file and environment | `-c`, `-t`, `-f` with `ini` or `env` |

`-t` accepts `all`, `base`, `become`, `cache`, `callback`, `cliconf`, `connection`, `httpapi`, `inventory`, `lookup`, `netconf`, `shell` and `vars`. `validate` checks base settings by default in core 2.21, and also installed plugins in core 2.22.

### ansible-test

Run it from inside `ansible_collections/<namespace>/<collection>/`.

```bash
ansible-test sanity --docker default              # Static checks
ansible-test units --docker default               # Unit tests
ansible-test integration --docker default -v      # Integration tests
ansible-test sanity --list-tests                  # List sanity tests
ansible-test env --show                           # Show the test environment
```

| Subcommand | Description |
| --- | --- |
| `sanity` | Static and code quality checks |
| `units` | Unit tests |
| `integration` | POSIX integration tests |
| `windows-integration` | Windows integration tests |
| `network-integration` | Network integration tests |
| `coverage` | Code coverage management and reporting |
| `env` | Show the test environment |
| `shell` | Open an interactive shell |

| Option | Description |
| --- | --- |
| `--docker`, `--venv`, `--local` | Choose where tests run |
| `--python` | Python version |
| `--requirements` | Install test requirements |
| `--changed`, `--base-branch` | Limit targets to changes |
| `--include`, `--exclude`, `--require` | Select targets |
| `--test`, `--skip-test`, `--list-tests` | Select sanity tests |
| `--coverage` | Measure code coverage |
| `--junit` | Write failures as JUnit XML |
| `--explain` | Show commands without running them |
| `--allow-destructive`, `--allow-root`, `--allow-disabled`, `--allow-unstable`, `--allow-unsupported` | Allow restricted integration tests |

### ansible-lint

```bash
ansible-lint                              # Lint the project
ansible-lint site.yml roles/web           # Lint specific paths
ansible-lint --profile production         # Apply a strict profile
ansible-lint --fix                        # Apply automatic fixes
ansible-lint -L                           # List rules
```

| Option | Description |
| --- | --- |
| `-L`, `--list-rules` | List rules |
| `-T`, `--list-tags` | List tags and their rules |
| `-P`, `--list-profiles` | List profiles |
| `--profile` | `min`, `basic`, `moderate`, `safety`, `shared` or `production` |
| `-f`, `--format` | `brief`, `full`, `md`, `json`, `codeclimate`, `quiet`, `pep8` or `sarif` |
| `--sarif-file` | Write SARIF output to a file |
| `-q` | Reduce verbosity; repeatable |
| `-v` | Raise verbosity |
| `-s`, `--strict` | Fail on warnings too |
| `--fix` | Apply fixes and reformat YAML; accepts rule ids or tags |
| `-t`, `--tags` | Check only these rules |
| `-x`, `--skip-list` | Skip these rules |
| `-w`, `--warn-list` | Report these rules as warnings |
| `--enable-list` | Enable optional rules |
| `-r`, `--rules-dir`, `-R` | Add custom rules; `-R` keeps the defaults |
| `-c`, `--config-file` | Config file |
| `-i`, `--ignore-file` | Ignore file |
| `--yamllint-file` | yamllint config file |
| `--generate-ignore` | Write `.ansible-lint-ignore` from current findings |
| `--exclude` | Exclude paths |
| `--project-dir` | Project directory |
| `--offline`, `--no-offline` | Control dependency installs |
| `--show-relpath` | Show paths relative to the current directory |
| `--nocolor`, `--force-color` | Control colored output |

### ansible-navigator

```bash
ansible-navigator run site.yml -m stdout           # Run with plain output
ansible-navigator run site.yml --ee false          # Run without a container
ansible-navigator doc ansible.builtin.copy -m stdout   # Read documentation
ansible-navigator replay artifact.json             # Review a past run
```

| Subcommand | Description |
| --- | --- |
| `run` | Run a playbook |
| `builder` | Build an execution environment |
| `collections` | Explore available collections |
| `config` | Explore the Ansible configuration |
| `doc` | Read module or plugin documentation |
| `exec` | Run a command inside an execution environment |
| `images` | Explore execution environment images |
| `inventory` | Explore an inventory |
| `lint` | Lint a file or directory |
| `replay` | Explore a past run from its artifact |
| `settings` | Review navigator settings |
| `welcome` | Open the welcome page |

| Option | Description |
| --- | --- |
| `--ee`, `--execution-environment` | Use an execution environment (`true` or `false`) |
| `--eei`, `--execution-environment-image` | Execution environment image |
| `--eev`, `--execution-environment-volume-mounts` | Bind mount a volume |
| `--ce`, `--container-engine` | `auto`, `podman`, `docker` or `container` |
| `--co`, `--container-options` | Extra container engine parameters |
| `-m`, `--mode` | `interactive` or `stdout` |
| `--lf`, `--log-file` | Navigator log file |
| `--la`, `--log-append` | Append to the log file |
| `--rad`, `--ansible-runner-artifact-dir` | Artifact directory |
| `--rac`, `--ansible-runner-rotate-artifacts-count` | Keep artifacts for the last n runs |
| `--rt`, `--ansible-runner-timeout` | Runner timeout |

### ansible-builder

```bash
ansible-builder build -t my-ee:1.0 -f execution-environment.yml   # Build an image
ansible-builder create -f execution-environment.yml               # Create a build context
ansible-builder introspect ./collections                          # List collection dependencies
```

| Option on `build` | Description |
| --- | --- |
| `-t`, `--tag` | Image name and tag |
| `-f`, `--file` | Execution environment definition |
| `-c`, `--context` | Build context directory |
| `--container-runtime` | `podman` or `docker` |
| `--build-arg` | Build argument |
| `--no-cache` | Build without cache |
| `--prune-images` | Remove dangling images afterwards |
| `--squash` | `new`, `all` or `off` |
| `--container-policy`, `--container-keyring` | Image signature verification |
| `--extra-build-cli-args` | Extra arguments for the build command |
| `--output-filename` | `Containerfile` or `Dockerfile` |
| `--galaxy-keyring`, `--galaxy-ignore-signature-status-codes`, `--galaxy-required-valid-signature-count` | Collection signature checks |
| `-v`, `--verbosity` | Verbosity level |

### molecule

```bash
molecule init scenario default     # Create a scenario
molecule test                      # Run the full test sequence
molecule converge                  # Apply the role and keep the instance
molecule login                     # Open a shell in an instance
```

| Command | Description |
| --- | --- |
| `init` | Create a new scenario |
| `test` | Run the full sequence |
| `create` | Start instances |
| `prepare` | Prepare instances |
| `converge` | Configure instances |
| `idempotence` | Converge twice and check for no changes |
| `side-effect` | Run side effect actions |
| `verify` | Run tests against instances |
| `check` | Run a dry run |
| `syntax` | Check syntax |
| `cleanup` | Clean up changes |
| `destroy` | Destroy instances |
| `dependency` | Manage dependencies |
| `list` | List instance status |
| `login` | Log in to an instance |
| `matrix` | List the test step sequence |
| `reset` | Reset temporary folders |
| `drivers` | List drivers |

## Inventory

An inventory lists the managed hosts and groups, as a static file, a script or a plugin. Pass several sources by repeating `-i`, or point `-i` at a directory.

### INI format

```ini
mail.example.com                  # Ungrouped host

[web]
web[01:20].example.com            # Numeric range
web-[a:f].example.com             # Alphabetic range
bastion ansible_host=10.0.0.5 ansible_port=2222 ansible_user=admin

[db]
db01 ansible_host=10.0.0.10

[web:vars]                        # Variables for a group
http_port=80

[prod:children]                   # Group of groups
web
db
```

### YAML format

```yaml
all:
  vars:
    ansible_user: admin
  hosts:
    mail.example.com:
  children:
    web:
      hosts:
        web[01:20].example.com:
      vars:
        http_port: 80
    db:
      hosts:
        db01:
          ansible_host: 10.0.0.10
    prod:
      children:
        web:
        db:
```

### Default groups and directories

| Name | Description |
| --- | --- |
| `all` | Every host |
| `ungrouped` | Hosts with no group other than `all` |
| `group_vars/<group>.yml` | Variables for a group; can also be a directory of files |
| `group_vars/all.yml` | Variables for every host |
| `host_vars/<host>.yml` | Variables for one host |

These directories are read next to the inventory and next to the playbook.

### Host patterns

| Pattern | Matches |
| --- | --- |
| `all` or `*` | All hosts |
| `web01` | One host |
| `web01,web02` | Several hosts; `:` also works |
| `web` | One group |
| `web:db` | Union of two groups |
| `web:&prod` | Intersection: in `web` and `prod` |
| `web:!db` | Exclusion: in `web`, not in `db` |
| `*.example.com` | Wildcard |
| `192.0.*` | Wildcard on an address |
| `~web\d+` | Regular expression |
| `web[0]` | First host of a group |
| `web[-1]` | Last host of a group |
| `web[0:2]` | Slice of a group |

Quote patterns in the shell, as in `'web:!db'`, because `!` and `&` are special characters.

### Behavioral parameters

| Variable | Description |
| --- | --- |
| `ansible_host` | Address to connect to |
| `ansible_port` | Connection port |
| `ansible_user` | Login user |
| `ansible_password` | Login password; keep it in Vault |
| `ansible_connection` | Connection plugin, such as `ssh`, `local`, `winrm` or `psrp` |
| `ansible_ssh_private_key_file` | SSH private key |
| `ansible_private_key_passphrase` | Private key passphrase |
| `ansible_ssh_common_args` | Extra arguments for sftp, scp and ssh |
| `ansible_ssh_extra_args` | Extra arguments for ssh |
| `ansible_sftp_extra_args` | Extra arguments for sftp |
| `ansible_scp_extra_args` | Extra arguments for scp |
| `ansible_ssh_pipelining` | Enable pipelining for this host |
| `ansible_ssh_executable` | Path to the ssh binary |
| `ansible_ssh_transfer_method` | File transfer method |
| `ansible_ssh_retries` | Connection retries |
| `ansible_ssh_timeout` | Connection timeout |
| `ansible_host_key_checking` | Check the SSH host key |
| `ansible_become` | Enable privilege escalation |
| `ansible_become_method` | Escalation method |
| `ansible_become_user` | Target user |
| `ansible_become_password` | Become password; keep it in Vault |
| `ansible_become_exe` | Escalation executable |
| `ansible_become_flags` | Escalation flags |
| `ansible_shell_type` | Target shell type |
| `ansible_shell_executable` | Target shell path |
| `ansible_python_interpreter` | Python path on the target |

### Inventory plugins

| Plugin | Description |
| --- | --- |
| `ini` | INI files |
| `yaml` | YAML files |
| `toml` | TOML files |
| `host_list` | Comma-separated host string |
| `advanced_host_list` | Host string with ranges |
| `script` | Executable that prints JSON |
| `auto` | Loads a plugin from the `plugin:` key in a YAML file |
| `constructed` | Builds groups and variables from existing data |
| `generator` | Builds hosts from a matrix of values |

An inventory script must support `--list` and `--host <name>`, and print JSON.

### Dynamic inventory example

The `auto` plugin reads the `plugin:` key. Cloud plugins come from collections, and the file name must end as each plugin requires, such as `azure_rm.yml` or `aws_ec2.yml`.

```yaml
# inventory/azure_rm.yml
plugin: azure.azcollection.azure_rm
auth_source: auto
include_vm_resource_groups:
  - rg-prod
keyed_groups:
  - prefix: tag
    key: tags
```

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1
keyed_groups:
  - key: tags.Role
    prefix: role
```

```yaml
# inventory/constructed.yml
plugin: ansible.builtin.constructed
strict: false
groups:
  debian_hosts: ansible_facts.os_family == 'Debian'
keyed_groups:
  - key: ansible_facts.distribution
    prefix: os
```

## Playbooks and tasks

A playbook is a YAML list of plays. A play maps hosts to tasks, and tasks run in order on each host.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  gather_facts: true
  vars:
    pkg: nginx
  tasks:
    - name: Install package
      ansible.builtin.package:
        name: "{{ pkg }}"
        state: present
      notify: Restart service

    - name: Enable service
      ansible.builtin.service:
        name: "{{ pkg }}"
        state: started
        enabled: true

  handlers:
    - name: Restart service
      ansible.builtin.service:
        name: "{{ pkg }}"
        state: restarted
```

### Play keywords

| Keyword | Description |
| --- | --- |
| `name` | Play name |
| `hosts` | Host pattern to target |
| `tasks`, `pre_tasks`, `post_tasks` | Task lists; order is `pre_tasks`, `roles`, `tasks`, `post_tasks` |
| `roles` | Roles to run |
| `handlers` | Handlers for this play |
| `vars`, `vars_files`, `vars_prompt` | Define, load or prompt for variables |
| `gather_facts`, `gather_subset`, `gather_timeout`, `fact_path` | Control fact gathering |
| `serial` | Batch size per rolling update: a number, a percentage or a list |
| `max_fail_percentage` | Abort when this share of a batch fails |
| `strategy` | Execution strategy |
| `order` | Host order: `inventory`, `reverse_inventory`, `sorted`, `reverse_sorted`, `shuffle` |
| `force_handlers` | Run handlers even after a failure |
| `collections` | Collection search list |
| `module_defaults` | Default module arguments |
| `validate_argspec` | Validate the play against an argument spec |

### Keywords for plays, blocks and tasks

| Keyword | Description |
| --- | --- |
| `any_errors_fatal` | Stop all hosts when one fails |
| `become`, `become_user`, `become_method`, `become_flags`, `become_exe` | Privilege escalation |
| `check_mode` | Force check mode on or off |
| `connection` | Connection plugin |
| `debugger` | Start the task debugger: `always`, `never`, `on_failed`, `on_unreachable`, `on_skipped` |
| `diff` | Force diff mode on or off |
| `environment` | Environment variables for tasks |
| `ignore_errors` | Continue after a task failure |
| `ignore_unreachable` | Continue after an unreachable host |
| `no_log` | Hide output and arguments |
| `port` | Connection port |
| `remote_user` | Login user |
| `run_once` | Run on one host in the batch |
| `tags` | Labels for selecting work |
| `throttle` | Limit concurrent workers |
| `timeout` | Task time limit in seconds |
| `vars` | Local variables |

### Task keywords

| Keyword | Description |
| --- | --- |
| `action` | Module to run; the usual form is the module name as the key |
| `args` | Module arguments as a mapping |
| `when` | Run only if the condition is true |
| `register` | Save the result to a variable |
| `notify` | Trigger handlers when the task changes something |
| `changed_when` | Define what counts as changed |
| `failed_when` | Define what counts as failed |
| `loop`, `with_<lookup>` | Repeat the task |
| `loop_control` | Tune the loop |
| `until`, `retries`, `delay` | Retry until a condition passes |
| `async`, `poll` | Run in the background |
| `delegate_to`, `delegate_facts` | Run on another host |
| `listen` | Handler topic name (handlers only) |

### Blocks and error handling

```yaml
- name: Deploy with rollback
  block:
    - name: Deploy
      ansible.builtin.command: /opt/deploy.sh
      register: deploy_result
      changed_when: deploy_result.rc == 0
      failed_when: deploy_result.rc not in [0, 2]
  rescue:
    - name: Roll back on failure
      ansible.builtin.command: /opt/rollback.sh
    - name: Show the failed task
      ansible.builtin.debug:
        var: ansible_failed_task.name
  always:
    - name: Always clean up
      ansible.builtin.file:
        path: /tmp/deploy
        state: absent
```

| Part | Description |
| --- | --- |
| `block` | Groups tasks and shares keywords with them |
| `rescue` | Runs when a task in `block` fails |
| `always` | Runs whatever the outcome |
| `ansible_failed_task` | Failed task, available in `rescue` |
| `ansible_failed_result` | Failed result, available in `rescue` |

### Conditionals

A `when` value is a raw Jinja expression, so do not wrap it in `{{ }}`. A list of conditions means all must be true.

```yaml
- name: Debian only
  ansible.builtin.debug:
    msg: Debian family
  when: ansible_facts['os_family'] == 'Debian'

- name: All must be true
  ansible.builtin.debug:
    msg: Both true
  when:
    - ansible_facts['distribution'] == 'Ubuntu'
    - ansible_facts['memtotal_mb'] > 1024

- name: Use a registered result
  ansible.builtin.debug:
    msg: The previous task failed
  when: result is failed

- name: Variable checks
  ansible.builtin.debug:
    msg: Defined and non-empty
  when: my_var is defined and my_var | length > 0
```

Since core 2.19, a condition must evaluate to a boolean. A non-empty string such as `when: my_string` fails, so write `my_string | length > 0`.

### Loops

```yaml
- name: Simple list
  ansible.builtin.user:
    name: "{{ item }}"
  loop: [alice, bob]

- name: List of mappings
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: wheel }
    - { name: bob, groups: users }

- name: Mapping
  ansible.builtin.debug:
    msg: "{{ item.key }}={{ item.value }}"
  loop: "{{ my_dict | dict2items }}"

- name: Loop control
  ansible.builtin.debug:
    msg: "{{ idx }}: {{ pkg }}"
  loop: "{{ packages }}"
  loop_control:
    loop_var: pkg
    index_var: idx
    label: "{{ pkg }}"
    pause: 1
    extended: true

- name: Retry until healthy
  ansible.builtin.uri:
    url: http://localhost/health
  register: health
  until: health.status == 200
  retries: 10
  delay: 5
```

| `loop_control` key | Description |
| --- | --- |
| `loop_var` | Rename `item`; needed in nested includes |
| `index_var` | Variable holding the index |
| `label` | Short text shown in output |
| `pause` | Seconds between iterations |
| `extended` | Enable `ansible_loop` |
| `extended_allitems` | Keep `ansible_loop.allitems` when extended |
| `break_when` | Stop the loop when true |

| `ansible_loop` key | Description |
| --- | --- |
| `index`, `index0` | Position from 1 or 0 |
| `revindex`, `revindex0` | Position from the end |
| `first`, `last` | True on the first or last item |
| `length` | Number of items |
| `previtem`, `nextitem` | Neighboring items |
| `allitems` | All items |

**Legacy `with_*` forms and their replacements**

| Legacy | Modern form |
| --- | --- |
| `with_items` | `loop` with the `flatten(levels=1)` filter |
| `with_list` | `loop` |
| `with_indexed_items` | `loop` with `index_var` |
| `with_dict` | `loop` with `dict2items` |
| `with_sequence` | `loop` with `range` |
| `with_fileglob` | `loop` with `lookup('fileglob', ...)` |
| `with_nested` | `loop` with the `product` filter |
| `with_subelements` | `loop` with the `subelements` filter |
| `with_together` | `loop` with the `zip` filter |
| `with_random_choice` | The `random` filter |

### Handlers

Handlers run once at the end of a play section, and only when notified.

```yaml
tasks:
  - name: Update config
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app.conf
    notify: Restart app

  - name: Run pending handlers now
    ansible.builtin.meta: flush_handlers

handlers:
  - name: Restart app
    ansible.builtin.service:
      name: app
      state: restarted
    listen: Restart app
```

### Includes and imports

| Static (parsed before the run) | Dynamic (evaluated at runtime) |
| --- | --- |
| `import_tasks` | `include_tasks` |
| `import_role` | `include_role` |
| `import_playbook` | No dynamic form |
| Tags and `when` apply to each imported task | Tags and `when` apply to the include itself |
| Tasks appear in `--list-tasks` | Tasks do not appear in `--list-tasks` |
| Cannot use loops | Can use loops |

### Async and background tasks

```yaml
- name: Long task, poll until done
  ansible.builtin.command: /opt/job.sh
  async: 1800      # Maximum runtime in seconds
  poll: 15         # Check every 15 seconds

- name: Fire and forget
  ansible.builtin.command: /opt/job.sh
  async: 3600
  poll: 0
  register: job

- name: Check it later
  ansible.builtin.async_status:
    jid: "{{ job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 10
```

### Delegation and local actions

```yaml
- name: Remove from load balancer
  ansible.builtin.command: /usr/bin/lb remove {{ inventory_hostname }}
  delegate_to: lb01

- name: Run on the control node
  ansible.builtin.command: date
  delegate_to: localhost

- name: Run once for the batch
  ansible.builtin.command: /opt/migrate.sh
  run_once: true
```

### Strategies and rolling updates

| Strategy | Description |
| --- | --- |
| `linear` | Default; hosts finish each task together |
| `free` | Each host runs as fast as it can |
| `host_pinned` | Like `free`, but a worker stays with one host until it finishes |
| `debug` | Linear with the interactive debugger |

```yaml
- name: Rolling update
  hosts: web
  serial: [1, 5, "25%"]      # Batches of 1, then 5, then 25%
  max_fail_percentage: 10
  any_errors_fatal: false
  strategy: linear
  tasks: []
```

### Tags

| Tag or option | Description |
| --- | --- |
| `always` | Runs unless skipped with `--skip-tags always` |
| `never` | Skips unless requested by tag |
| `--tags tagged` | Run only tagged tasks |
| `--tags untagged` | Run only untagged tasks |
| `--tags all` | Run everything; the default |

### meta actions

| Action | Description |
| --- | --- |
| `flush_handlers` | Run notified handlers now |
| `refresh_inventory` | Reload the inventory |
| `noop` | Do nothing |
| `clear_facts` | Clear facts for the host |
| `clear_host_errors` | Clear failed and unreachable status |
| `end_play` | End the play for all hosts |
| `end_host` | End the play for this host |
| `end_batch` | End the current batch |
| `reset_connection` | Close the connection so the next task reconnects |

`end_role` is also available, and ends the current role for the host.

## Variables

Variables come from many sources, and extra vars (`-e`) always win. Reference them with `{{ name }}`, and quote the whole YAML value when it starts with `{{`.

### Define variables

```yaml
- name: Variable sources in a play
  hosts: all
  vars:
    app_port: 8080
    users:
      - alice
      - bob
    db:
      host: db01
      port: 5432
  vars_files:
    - vars/common.yml
    - "vars/{{ ansible_facts['os_family'] }}.yml"
  vars_prompt:
    - name: release
      prompt: Which release?
      default: main
      private: false
    - name: admin_password
      prompt: Admin password
      private: true
      confirm: true
      encrypt: sha512_crypt
      salt_size: 16
  tasks:
    - name: Set a variable at runtime
      ansible.builtin.set_fact:
        full_url: "http://{{ db.host }}:{{ db.port }}"
        cacheable: true

    - name: Load variables from a file
      ansible.builtin.include_vars:
        file: vars/extra.yml
        name: extra

    - name: Save a task result
      ansible.builtin.command: whoami
      register: who
      changed_when: false

    - name: Use it
      ansible.builtin.debug:
        msg: "{{ who.stdout }} on port {{ app_port }}"
```

Access nested values with `db.host` or `db['host']`. The bracket form is safer, because it cannot clash with Python method names.

### Registered result fields

| Field | Description |
| --- | --- |
| `changed` | True if the task changed something |
| `failed` | True if the task failed |
| `skipped` | True if the task was skipped |
| `rc` | Return code |
| `stdout`, `stdout_lines` | Standard output as text or a list |
| `stderr`, `stderr_lines` | Standard error as text or a list |
| `msg` | Message from the module |
| `results` | One result per item when the task used a loop |

### Variable precedence

The list runs from lowest to highest. The later entry wins.

1. Command line values, such as `-u user` (these are not variables)
2. Role defaults (`defaults/main.yml`)
3. Inventory file or script group vars
4. Inventory `group_vars/all`
5. Playbook `group_vars/all`
6. Inventory `group_vars/*`
7. Playbook `group_vars/*`
8. Inventory file or script host vars
9. Inventory `host_vars/*`
10. Playbook `host_vars/*`
11. Host facts and cached `set_fact`
12. Play `vars`
13. Play `vars_prompt`
14. Play `vars_files`
15. Role vars (`vars/main.yml`)
16. Block vars
17. Task vars
18. `include_vars`
19. `set_fact` and registered vars
20. Role and `include_role` params
21. Include params
22. Extra vars (`-e`)

### Magic variables

| Variable | Description |
| --- | --- |
| `inventory_hostname` | Current host name from the inventory |
| `inventory_hostname_short` | Host name up to the first dot |
| `inventory_dir` | Directory of the inventory source for the host |
| `inventory_file` | File of the inventory source for the host |
| `group_names` | Groups the current host belongs to |
| `groups` | All groups and their hosts |
| `hostvars` | Variables of every host |
| `omit` | Omits a module option, as in `default(omit)` |
| `playbook_dir` | Directory of the current playbook |
| `role_name` | Name of the running role |
| `role_path` | Path of the running role |
| `ansible_role_name` | Fully qualified role name |
| `ansible_role_names` | Roles in the play, including dependencies |
| `ansible_play_role_names` | Roles imported directly into the play |
| `ansible_dependent_role_names` | Roles imported as dependencies |
| `ansible_parent_role_names` | Chain of roles that included this one |
| `ansible_parent_role_paths` | Paths of those parent roles |
| `ansible_play_name` | Name of the current play |
| `ansible_play_hosts` | Active hosts in the play |
| `ansible_play_batch` | Active hosts in the current `serial` batch |
| `ansible_play_hosts_all` | All hosts targeted by the play |
| `ansible_check_mode` | True in check mode |
| `ansible_diff_mode` | True in diff mode |
| `ansible_forks` | Maximum forks for the run |
| `ansible_limit` | Value of `--limit` |
| `ansible_run_tags` | Value of `--tags` |
| `ansible_skip_tags` | Value of `--skip-tags` |
| `ansible_inventory_sources` | Inventory sources in use |
| `ansible_config_file` | Config file in use |
| `ansible_collection_name` | Collection of the running task |
| `ansible_playbook_python` | Python path on the control node |
| `ansible_search_path` | Search path for relative files |
| `ansible_verbosity` | Current verbosity |
| `ansible_version` | Version details: `full`, `major`, `minor`, `revision`, `string` |
| `ansible_loop` | Extended loop details |
| `ansible_loop_var` | Name set by `loop_var` |
| `ansible_index_var` | Name set by `index_var` |
| `ansible_facts` | Facts for the current host |
| `ansible_local` | Custom local facts |

`play_hosts` and `role_names` are deprecated aliases of `ansible_play_batch` and `ansible_play_role_names`.

### Facts

```bash
ansible host -m ansible.builtin.setup                               # All facts
ansible host -m ansible.builtin.setup -a 'filter=ansible_*_mb'      # Filter by name
ansible host -m ansible.builtin.setup -a 'gather_subset=network'    # Limit by subset
```

```yaml
- name: Read facts
  hosts: all
  gather_facts: true
  gather_subset:
    - min
    - network
    - "!hardware"
  tasks:
    - name: Show common facts
      ansible.builtin.debug:
        msg: >-
          {{ ansible_facts['hostname'] }}
          {{ ansible_facts['distribution'] }}
          {{ ansible_facts['distribution_version'] }}
          {{ ansible_facts['default_ipv4']['address'] }}
```

| Fact | Description |
| --- | --- |
| `hostname`, `fqdn` | Host names |
| `os_family`, `distribution`, `distribution_version`, `distribution_major_version` | Operating system |
| `architecture`, `kernel` | Platform |
| `processor_vcpus`, `processor_cores` | CPU |
| `memtotal_mb`, `memfree_mb` | Memory |
| `default_ipv4`, `all_ipv4_addresses`, `interfaces` | Network |
| `mounts`, `devices` | Storage |
| `date_time` | Time at fact gathering |
| `env` | Environment variables |
| `user_id`, `user_dir` | Connection user |
| `pkg_mgr`, `service_mgr` | System tooling |
| `virtualization_type`, `virtualization_role` | Virtualization |

Inside `ansible_facts`, the `ansible_` prefix is dropped. Use `ansible_facts['distribution']`, not `ansible_distribution`: the injected form is deprecated and `INJECT_FACTS_AS_VARS` is scheduled to default to `False` in core 2.24.

**Custom facts**: put a `.fact` file of INI, JSON or an executable that prints JSON in `/etc/ansible/facts.d/` on the target. Read it through `ansible_local`, as in `ansible_local['app']['general']['version']`.

**Facts helpers**: `package_facts`, `service_facts` and `mount_facts` add package, service and mount data.

**Fact caching**: set `fact_caching = jsonfile` and `fact_caching_connection` in `ansible.cfg` to reuse facts between runs.

### YAML tags and multi-line values

| Syntax | Description |
| --- | --- |
| `!vault` | Inline Vault-encrypted value |
| `!unsafe` | Mark a value so it is never templated |
| `key: >` | Folded block; newlines become spaces |
| `key: >-` | Folded block without the final newline |
| `key: \|` | Literal block; newlines are kept |
| `key: \|-` | Literal block without the final newline |
| `&name`, `*name`, `<<:` | YAML anchors, aliases and merge keys |

## Templates, filters, tests and lookups

Ansible uses Jinja2 on the control node. Filters change data, tests return true or false, and lookups read data from outside sources.

### Jinja2 syntax

```jinja
{# A comment #}
Server: {{ inventory_hostname }}
{{ my_var | default('none') }}

{% if env == 'prod' %}
log_level = warn
{% elif env == 'stage' %}
log_level = info
{% else %}
log_level = debug
{% endif %}

{% for host in groups['web'] %}
server {{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }}:80;
{% endfor %}

{% for item in items_list if item.enabled %}
{{ loop.index }} {{ item.name }}{{ ',' if not loop.last }}
{% else %}
# No items
{% endfor %}

{%- set total = 10 -%}
{% macro row(k, v) %}{{ k }}={{ v }}{% endmacro %}
{{ row('a', 1) }}
{% include 'other.j2' %}
```

| Syntax | Description |
| --- | --- |
| `{{ }}` | Print an expression |
| `{% %}` | Run a statement |
| `{# #}` | Comment |
| `{%- -%}` | Trim whitespace around the tag |
| `~` | Join values as text |
| `in`, `not in` | Membership |
| `and`, `or`, `not` | Logic |
| `a if cond else b` | Inline condition |
| `range(n)` | Sequence of numbers |
| `lipsum`, `dict`, `namespace` | Jinja global functions |
| `now()` | Current time; `now(utc=true, fmt='%H')` formats it |
| `undef()` | Create an undefined value with a hint |

### Filter examples

```yaml
# Defaults and required values
- "{{ name | default('guest') }}"              # Fallback value
- "{{ name | default('guest', true) }}"        # Also replaces empty values
- "{{ path | default(omit) }}"                 # Omit the module option
- "{{ token | mandatory }}"                    # Fail if undefined

# Lists
- "{{ [1, 2, 2] | unique }}"                   # Remove duplicates
- "{{ [3, 1] | sort }}"                        # Sort
- "{{ [[1], [2, [3]]] | flatten }}"            # Flatten
- "{{ a | union(b) }}"                         # Set union
- "{{ a | intersect(b) }}"                     # Set intersection
- "{{ a | difference(b) }}"                    # Items in a, not in b
- "{{ a | symmetric_difference(b) }}"          # Items in only one list
- "{{ a | zip(b) | list }}"                    # Pair items
- "{{ users | map(attribute='name') | list }}"            # Extract a field
- "{{ users | selectattr('active') | list }}"             # Keep by attribute
- "{{ users | rejectattr('age', 'lt', 18) | list }}"      # Drop by attribute
- "{{ nums | select('odd') | list }}"                     # Keep by test
- "{{ users | groupby('dept') }}"                         # Group by a field
- "{{ nums | sum }}"                                      # Add values
- "{{ nums | min }}"                                      # Smallest value

# Dictionaries
- "{{ a | combine(b) }}"                                  # Merge
- "{{ a | combine(b, recursive=true) }}"                  # Deep merge
- "{{ a | dict2items }}"                                  # Dict to list of key and value
- "{{ a | items2dict }}"                                  # List of key and value to dict
- "{{ keys | zip(values) | community.general.dict }}"     # Build from pairs

# Strings
- "{{ text | regex_replace('^a', 'b') }}"      # Replace by regex
- "{{ text | regex_search('v(\\d+)', '\\1') }}"          # First regex match
- "{{ text | regex_findall('\\d+') }}"         # All regex matches
- "{{ text | replace('a', 'b') }}"             # Plain replace
- "{{ text | split(',') }}"                    # Split into a list
- "{{ items | join(', ') }}"                   # Join a list
- "{{ text | trim | lower }}"                  # Chain filters
- "{{ value | quote }}"                        # Quote for the shell
- "{{ flag | ternary('yes', 'no') }}"          # Choose by a boolean

# Formats
- "{{ data | to_nice_json(indent=2) }}"        # Readable JSON
- "{{ data | to_nice_yaml }}"                  # Readable YAML
- "{{ text | from_json }}"                     # Parse JSON
- "{{ text | from_yaml }}"                     # Parse YAML

# Paths, encoding and hashing
- "{{ '/etc/app.conf' | basename }}"           # app.conf
- "{{ '/etc/app.conf' | dirname }}"            # /etc
- "{{ ['/etc', 'app'] | path_join }}"          # /etc/app
- "{{ text | b64encode }}"                     # Base64 encode
- "{{ text | hash('sha256') }}"                # Hash
- "{{ pw | password_hash('sha512') }}"         # Password hash for the user module
```

### Filter reference

| Group | Filter | Description |
| --- | --- | --- |
| Default | `default`, `d` | Fallback for undefined values |
| Default | `mandatory` | Fail if undefined |
| Type | `bool`, `int`, `float`, `string`, `list` | Convert the type |
| Type | `type_debug` | Show the Python type |
| Text | `lower`, `upper`, `capitalize`, `title` | Change case |
| Text | `trim`, `center`, `indent`, `truncate` | Layout |
| Text | `wordcount`, `wordwrap` | Word handling |
| Text | `replace`, `format`, `split`, `join` | Edit and join |
| Text | `regex_replace`, `regex_search`, `regex_findall`, `regex_escape` | Regular expressions |
| Text | `comment` | Wrap text as a comment |
| Text | `quote`, `escape`, `e`, `forceescape`, `safe`, `striptags`, `urlencode`, `urldecode`, `urlize`, `xmlattr` | Escape and encode |
| List | `first`, `last`, `length`, `count` | Pick or count |
| List | `min`, `max`, `sum`, `sort`, `reverse`, `unique`, `shuffle` | Order and aggregate |
| List | `flatten`, `batch`, `slice`, `zip`, `zip_longest` | Reshape |
| List | `union`, `intersect`, `difference`, `symmetric_difference` | Set operations |
| List | `product`, `permutations`, `combinations`, `subelements` | Combine elements |
| List | `map`, `select`, `reject`, `selectattr`, `rejectattr`, `groupby`, `attr`, `extract` | Filter and extract |
| Dict | `combine`, `dict2items`, `items2dict`, `dictsort`, `items`, `rekey_on_member` | Reshape mappings |
| Math | `abs`, `round`, `pow`, `root`, `log` | Arithmetic |
| Math | `human_readable`, `human_to_bytes`, `filesizeformat` | Convert sizes |
| Random | `random`, `to_uuid` | Generate values |
| Format | `to_json`, `tojson`, `to_nice_json`, `from_json` | JSON |
| Format | `to_yaml`, `to_nice_yaml`, `from_yaml`, `from_yaml_all` | YAML |
| Format | `pprint` | Pretty print |
| Encoding | `b64encode`, `b64decode` | Base64 |
| Hash | `hash`, `md5`, `sha1`, `checksum`, `password_hash` | Hashes |
| Path | `basename`, `dirname`, `expanduser`, `expandvars`, `realpath`, `relpath`, `splitext`, `normpath`, `commonpath`, `path_join`, `fileglob` | POSIX paths |
| Path | `win_basename`, `win_dirname`, `win_splitdrive` | Windows paths |
| URL | `urlsplit` | Split a URL into parts |
| Time | `strftime`, `to_datetime` | Dates and times |
| Logic | `ternary` | Choose between two values |
| Vault | `vault`, `unvault` | Encrypt or decrypt a value |

**Useful filters from collections**: `ansible.utils.ipaddr`, `ansible.utils.ipsubnet`, `community.general.json_query`, `community.general.to_time_unit`, `ansible.utils.from_xml` and `ansible.utils.to_xml`.

### Tests

Use a test with `is`, as in `when: result is failed`.

| Group | Test | Description |
| --- | --- | --- |
| Variable | `defined`, `undefined`, `none` | Existence |
| Type | `string`, `number`, `integer`, `float`, `boolean`, `sequence`, `mapping`, `iterable`, `callable` | Data type |
| Value | `true`, `false`, `truthy`, `falsy` | Boolean value |
| Compare | `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `sameas` | Compare values |
| Number | `even`, `odd`, `divisibleby`, `nan` | Number checks |
| Text | `lower`, `upper`, `match`, `search`, `regex`, `escaped` | Text checks |
| Collection | `in`, `contains`, `subset`, `superset`, `all`, `any` | Membership |
| Version | `version` | Compare versions, as in `v is version('2.0', '>=')` |
| Result | `success`, `succeeded`, `failed`, `failure`, `changed`, `change`, `skipped`, `skip`, `unreachable`, `reachable`, `finished`, `started`, `timedout` | Task status |
| Path | `file`, `directory`, `link`, `exists`, `mount`, `abs`, `same_file`, `link_exists` | Paths on the control node |
| URI | `uri`, `url`, `urn` | Address format |
| Vault | `vault_encrypted`, `vaulted_file` | Encrypted data |

### Lookups

Lookups run on the control node. `lookup()` returns text, and `query()` or `q()` returns a list.

```yaml
- name: Lookup examples
  ansible.builtin.debug:
    msg:
      - "{{ lookup('ansible.builtin.env', 'HOME') }}"
      - "{{ lookup('ansible.builtin.file', '/etc/hostname') }}"
      - "{{ lookup('ansible.builtin.pipe', 'date') }}"
      - "{{ lookup('ansible.builtin.password', '/dev/null length=20') }}"
      - "{{ query('ansible.builtin.fileglob', '/etc/*.conf') }}"
      - "{{ lookup('ansible.builtin.vars', 'my_var', default='none') }}"
```

| Lookup | Description |
| --- | --- |
| `config` | Read a resolved Ansible setting |
| `csvfile` | Read from a CSV or TSV file |
| `dict` | Return key and value items from a dictionary |
| `env` | Read an environment variable |
| `file` | Read file contents |
| `fileglob` | List files matching a pattern |
| `first_found` | Return the first file found |
| `indexed_items` | Return items with their index |
| `ini` | Read from an INI file |
| `inventory_hostnames` | List hosts matching a pattern |
| `items` | Return a list of items |
| `lines` | Read lines from a command |
| `list` | Return what it is given |
| `nested` | Build nested combinations of lists |
| `password` | Retrieve or generate a random password |
| `pipe` | Read output from a command |
| `random_choice` | Return a random element |
| `sequence` | Generate a number sequence |
| `subelements` | Traverse a nested key |
| `template` | Return a rendered template |
| `together` | Merge lists into a synchronized list |
| `unvault` | Read vaulted file contents |
| `url` | Return contents from a URL |
| `varnames` | Find matching variable names |
| `vars` | Read a templated variable value |

## Modules by category

`ansible.builtin` has 71 modules, all listed here. The ansible 14.5.0 package bundles about 8,800 more in collections, so this page lists the common ones: run `ansible-doc -l` for your full set.

Always use the fully qualified name, such as `ansible.builtin.copy`.

### Built-in modules

| Category | Module | Description |
| --- | --- | --- |
| Command | `command` | Run a command without a shell |
| Command | `shell` | Run a command through a shell |
| Command | `raw` | Run over SSH without Python on the target |
| Command | `script` | Copy a local script and run it |
| Command | `expect` | Answer interactive prompts |
| File | `file` | Manage files, directories, links and permissions |
| File | `copy` | Copy files to the target |
| File | `template` | Render a Jinja2 template to the target |
| File | `fetch` | Copy files from the target |
| File | `stat` | Read file status |
| File | `find` | Search for files |
| File | `lineinfile` | Manage one line in a file |
| File | `blockinfile` | Manage a marked block in a file |
| File | `replace` | Replace all regex matches in a file |
| File | `assemble` | Build a file from fragments |
| File | `slurp` | Read a remote file as base64 |
| File | `tempfile` | Create a temporary file or directory |
| File | `unarchive` | Unpack an archive |
| File | `get_url` | Download a file |
| Package | `package` | Use the system package manager |
| Package | `apt` | Manage apt packages |
| Package | `apt_key` | Manage apt keys |
| Package | `apt_repository` | Manage apt repositories |
| Package | `deb822_repository` | Manage repositories in deb822 format |
| Package | `debconf` | Configure a deb package |
| Package | `dpkg_selections` | Set dpkg package selections |
| Package | `dnf` | Manage packages with dnf |
| Package | `dnf5` | Manage packages with dnf5 |
| Package | `rpm_key` | Manage rpm keys |
| Package | `yum_repository` | Manage yum and dnf repositories |
| Package | `pip` | Manage Python packages |
| Service | `service` | Manage services on any init system |
| Service | `systemd_service` | Manage systemd units |
| Service | `systemd` | Alias of `systemd_service` |
| Service | `sysvinit` | Manage SysV services |
| System | `user` | Manage users |
| System | `group` | Manage groups |
| System | `hostname` | Set the host name |
| System | `cron` | Manage cron jobs |
| System | `mount_facts` | Gather mount information |
| System | `getent` | Read Unix databases |
| System | `known_hosts` | Manage SSH known hosts |
| System | `iptables` | Manage iptables rules |
| System | `reboot` | Reboot and wait for the host |
| Network | `uri` | Call web services |
| Network | `wait_for` | Wait for a port, file or condition |
| Network | `wait_for_connection` | Wait until the host is reachable |
| Network | `ping` | Test the Ansible connection, not ICMP |
| Source | `git` | Manage git checkouts |
| Source | `subversion` | Manage Subversion checkouts |
| Facts | `setup` | Gather facts |
| Facts | `gather_facts` | Run the configured fact modules |
| Facts | `package_facts` | Gather package information |
| Facts | `service_facts` | Gather service information |
| Flow | `debug` | Print messages |
| Flow | `assert` | Check conditions |
| Flow | `fail` | Fail with a message |
| Flow | `pause` | Pause the run |
| Flow | `meta` | Run an internal Ansible action |
| Flow | `async_status` | Check a background task |
| Flow | `validate_argument_spec` | Validate arguments against a spec |
| Variables | `set_fact` | Set host variables |
| Variables | `set_stats` | Set custom run statistics |
| Variables | `include_vars` | Load variables from a file |
| Inventory | `add_host` | Add a host to the in-memory inventory |
| Inventory | `group_by` | Create groups from values |
| Reuse | `import_playbook` | Import a playbook |
| Reuse | `import_tasks` | Import a task file |
| Reuse | `import_role` | Import a role |
| Reuse | `include_tasks` | Include a task file at runtime |
| Reuse | `include_role` | Include a role at runtime |

### Common module options

```yaml
- name: Create a directory
  ansible.builtin.file:
    path: /opt/app
    state: directory          # file, directory, link, hard, touch or absent
    owner: app
    group: app
    mode: "0750"
    recurse: true

- name: Copy content with a backup and validation
  ansible.builtin.copy:
    content: "key=value\n"
    dest: /etc/app.conf
    backup: true
    validate: /usr/bin/app --check %s

- name: Edit one line
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: PermitRootLogin no
    state: present

- name: Call an API
  ansible.builtin.uri:
    url: https://api.example.com/items
    method: POST
    body_format: json
    body:
      name: demo
    headers:
      Authorization: "Bearer {{ api_token }}"
    status_code: [200, 201]
    return_content: true
  register: api_response

- name: Make a command idempotent
  ansible.builtin.command:
    cmd: /opt/init.sh
    creates: /opt/.initialized
    chdir: /opt
```

### Modules from collections

| Area | Collection | Common modules |
| --- | --- | --- |
| POSIX | `ansible.posix` | `acl`, `at`, `authorized_key`, `firewalld`, `mount`, `patch`, `seboolean`, `selinux`, `synchronize`, `sysctl` |
| General | `community.general` | `archive`, `git_config`, `homebrew`, `ini_file`, `make`, `nmcli`, `npm`, `pacman`, `snap`, `timezone`, `ufw`, `xml`, `zypper` |
| Crypto | `community.crypto` | `acme_certificate`, `openssl_csr`, `openssl_privatekey`, `openssh_keypair`, `x509_certificate` |
| Windows | `ansible.windows` | `win_command`, `win_copy`, `win_dsc`, `win_feature`, `win_file`, `win_package`, `win_reboot`, `win_service`, `win_shell`, `win_template`, `win_updates`, `win_user` |
| Windows | `community.windows` | `win_firewall_rule`, `win_scheduled_task`, `win_lineinfile` |
| Azure | `azure.azcollection` | `azure_rm_resourcegroup`, `azure_rm_virtualmachine`, `azure_rm_virtualnetwork`, `azure_rm_storageaccount`, `azure_rm_aks`, `azure_rm_keyvault` |
| AWS | `amazon.aws` | `ec2_instance`, `ec2_vpc_net`, `s3_bucket`, `s3_object`, `iam_role` |
| Google Cloud | `google.cloud` | `gcp_compute_instance`, `gcp_compute_network`, `gcp_storage_bucket` |
| Containers | `community.docker` | `docker_container`, `docker_image`, `docker_network`, `docker_compose_v2` |
| Containers | `containers.podman` | `podman_container`, `podman_image` |
| Kubernetes | `kubernetes.core` | `k8s`, `k8s_info`, `k8s_exec`, `helm`, `helm_repository` |
| VMware | `community.vmware` | `vmware_guest`, `vmware_host`, `vmware_vm_info` |
| Databases | `ansible.mysql` | `mysql_db`, `mysql_user`, `mysql_query` |
| Databases | `community.postgresql` | `postgresql_db`, `postgresql_user`, `postgresql_query` |
| Databases | `community.mongodb` | `mongodb_user`, `mongodb_replicaset` |
| Networking | `ansible.netcommon` | `cli_command`, `cli_config`, `net_get`, `net_put`, `netconf_config` |
| Networking | `cisco.ios` | `ios_config`, `ios_command`, `ios_facts`, `ios_vlans` |
| Networking | `arista.eos`, `cisco.nxos` | `eos_config`, `nxos_config` and similar |
| Secrets | `community.hashi_vault` | `vault_kv2_get`, `vault_write`, `vault_read` |
| Utilities | `ansible.utils` | `validate`, `fact_diff`, `cli_parse` |

## Roles and collections

A role packages tasks, variables and files for reuse. A collection packages roles, modules and plugins for distribution.

### Project layout

```text
project/
├── ansible.cfg
├── site.yml
├── requirements.yml
├── inventories/
│   ├── prod/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   └── host_vars/
│   └── staging/
├── roles/
├── library/            # Custom modules
├── module_utils/       # Shared module code
├── filter_plugins/     # Custom filters
└── collections/
```

### Role layout

```text
roles/web/
├── defaults/main.yml         # Lowest precedence variables
├── vars/main.yml             # High precedence variables
├── tasks/main.yml            # Entry point
├── handlers/main.yml         # Handlers
├── templates/                # Jinja2 templates
├── files/                    # Static files
├── meta/main.yml             # Dependencies and Galaxy data
├── meta/argument_specs.yml   # Argument validation
├── library/                  # Role modules
├── filter_plugins/           # Role filters
├── tests/                    # Test playbooks
└── README.md
```

### Use roles

```yaml
- name: Three ways to use a role
  hosts: web
  roles:
    - common                      # Static, runs before tasks
    - role: web
      vars:
        port: 8080
      tags: [web]
      when: ansible_facts['os_family'] == 'Debian'
  tasks:
    - name: Static import
      ansible.builtin.import_role:
        name: web
        tasks_from: install

    - name: Dynamic include
      ansible.builtin.include_role:
        name: web
        tasks_from: configure
        defaults_from: main
        vars_from: main
        handlers_from: main
        public: true
        apply:
          tags: [web]
```

### Role metadata and argument specs

```yaml
# roles/web/meta/main.yml
galaxy_info:
  author: your_name
  description: Web server role
  license: MIT
  min_ansible_version: "2.15"
dependencies:
  - role: common
    vars:
      level: 1
allow_duplicates: false
```

```yaml
# roles/web/meta/argument_specs.yml
argument_specs:
  main:
    short_description: Configure the web server
    options:
      port:
        type: int
        required: false
        default: 80
        description: Listening port
```

### requirements.yml

```yaml
roles:
  - name: geerlingguy.nginx
    version: 3.2.0
  - name: my_role
    src: https://github.com/org/my_role.git
    scm: git
    version: main

collections:
  - name: community.general
    version: ">=10.0.0,<12.0.0"
  - name: https://github.com/org/my_collection.git
    type: git
    version: main
  - name: ./local/my_ns-my_col-1.0.0.tar.gz
    type: file
  - name: my_ns.my_col
    source: https://hub.example.com/api/galaxy/
```

```bash
ansible-galaxy install -r requirements.yml     # Install roles and collections
```

### Collection layout

```text
my_ns/my_col/
├── galaxy.yml         # Name, version, dependencies
├── README.md
├── meta/runtime.yml   # Required core version and redirects
├── plugins/
│   ├── modules/
│   ├── module_utils/
│   ├── filter/
│   ├── lookup/
│   ├── inventory/
│   └── callback/
├── roles/
├── playbooks/
├── docs/
└── tests/
```

```yaml
- name: Use collection content
  hosts: all
  collections:
    - community.general
  tasks:
    - name: Fully qualified name (preferred)
      community.general.timezone:
        name: Africa/Cairo

    - name: Role from a collection
      ansible.builtin.import_role:
        name: my_ns.my_col.my_role

- name: Playbook from a collection
  ansible.builtin.import_playbook: my_ns.my_col.my_playbook
```

Collections are searched in `collections_path`, by default `~/.ansible/collections` and `/usr/share/ansible/collections`, and in `collections/` next to the playbook.

### Execution environments

An execution environment is a container image with Ansible, collections and dependencies. Build one with `ansible-builder`, and run it with `ansible-navigator`.

```yaml
# execution-environment.yml
version: 3
images:
  base_image:
    name: quay.io/fedora/fedora:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt
```

## Vault

Vault encrypts files or single values with AES256, so secrets can live in source control. Do not commit the password file.

### Encrypted files

```bash
ansible-vault create secrets.yml                 # Create and open in the editor
ansible-vault edit secrets.yml                   # Edit in place
ansible-vault view secrets.yml                   # Print the contents
ansible-vault encrypt vars.yml                   # Encrypt an existing file
ansible-vault encrypt a.yml --output a.enc.yml   # Write to a new file
ansible-vault decrypt secrets.yml                # Decrypt in place
ansible-vault decrypt secrets.yml --output -     # Print to the screen
ansible-vault rekey secrets.yml                  # Change the password
```

### Encrypted strings

```bash
ansible-vault encrypt_string 'S3cret' --name 'db_password'       # Encrypt a value
ansible-vault encrypt_string -p --name 'db_password'             # Prompt for the value
printf 'S3cret' | ansible-vault encrypt_string --stdin-name 'db_password'  # From stdin
```

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3435...
```

### Provide the password

```bash
ansible-playbook site.yml -J                                  # Prompt
ansible-playbook site.yml --vault-password-file ~/.vault_pass # From a file or script
ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass ansible-playbook site.yml   # From the environment
```

If the password file is executable, Ansible runs it and reads the password from its output. This lets you fetch it from a secret manager.

### Vault IDs

A vault ID labels a password, which lets one project use several.

```bash
ansible-vault create --vault-id dev@prompt dev.yml                 # Label dev, prompt
ansible-vault encrypt --vault-id prod@~/.vault_prod prod.yml       # Label prod, from a file
ansible-vault encrypt_string --vault-id dev@prompt --name x 'val'  # Label a string
ansible-vault rekey --vault-id old@prompt --new-vault-id new@prompt f.yml   # Move to a new ID
ansible-playbook site.yml --vault-id dev@prompt --vault-id prod@~/.vault_prod   # Use both
```

| Form | Description |
| --- | --- |
| `label@prompt` | Prompt for the password |
| `label@file` | Read the password from a file |
| `label@script` | Run a script for the password |
| `@prompt` | Prompt, with no label |

```ini
# ansible.cfg
[defaults]
vault_identity_list = dev@~/.vault_dev, prod@~/.vault_prod
```

A common layout keeps plain and encrypted values apart, and the plain file points at the secret. Use `group_vars/web/vars.yml` with `db_password: "{{ vault_db_password }}"`, and put `vault_db_password` in `group_vars/web/vault.yml`.

## Plugins

Plugins extend Ansible on the control node, while modules run on the target. List any type with `ansible-doc -t <type> -l`.

| Type | Local directory | Purpose | In `ansible.builtin` |
| --- | --- | --- | --- |
| `action` | `action_plugins/` | Run control node logic before a module | Used by `template`, `copy` and others |
| `become` | `become_plugins/` | Privilege escalation | `sudo`, `su`, `runas` |
| `cache` | `cache_plugins/` | Store facts and inventory data | `memory`, `jsonfile` |
| `callback` | `callback_plugins/` | React to events and format output | `default`, `minimal`, `oneline`, `tree`, `junit` |
| `cliconf` | `cliconf_plugins/` | CLI interface to network devices | None; see network collections |
| `connection` | `connection_plugins/` | Reach the target | `ssh`, `local`, `winrm`, `psrp` |
| `doc_fragments` | `doc_fragments/` | Shared documentation | Various |
| `filter` | `filter_plugins/` | Transform data | See the filter reference |
| `httpapi` | `httpapi_plugins/` | HTTP interface to network devices | None; see network collections |
| `inventory` | `inventory_plugins/` | Load hosts | See the inventory plugins |
| `lookup` | `lookup_plugins/` | Read outside data | See the lookups |
| `module` | `library/` | Do work on the target | See the modules |
| `module_utils` | `module_utils/` | Shared module code | Various |
| `netconf` | `netconf_plugins/` | NETCONF interface to network devices | None; see network collections |
| `shell` | `shell_plugins/` | Format commands for the target shell | `sh`, `powershell`, `cmd` |
| `strategy` | `strategy_plugins/` | Control task execution | `linear`, `free`, `host_pinned`, `debug` |
| `terminal` | `terminal_plugins/` | Handle network device terminals | None; see network collections |
| `test` | `test_plugins/` | Evaluate data | See the tests |
| `vars` | `vars_plugins/` | Add variables | `host_group_vars` |

### Connection plugins

| Plugin | Description |
| --- | --- |
| `ansible.builtin.ssh` | OpenSSH; the default |
| `ansible.builtin.local` | The control node |
| `ansible.builtin.winrm` | Windows through WinRM |
| `ansible.builtin.psrp` | Windows through PowerShell Remoting |
| `ansible.netcommon.network_cli` | Network device CLI |
| `ansible.netcommon.netconf` | NETCONF |
| `ansible.netcommon.httpapi` | Device HTTP API |
| `ansible.netcommon.libssh` | SSH through libssh |
| `amazon.aws.aws_ssm` | AWS Systems Manager |
| `community.docker.docker` | Docker containers |
| `containers.podman.podman` | Podman containers |
| `kubernetes.core.kubectl` | Kubernetes pods |
| `community.general.chroot` | Chroot directories |
| `community.general.lxc`, `community.general.lxd` | Linux containers |
| `community.general.wsl` | Windows Subsystem for Linux |
| `community.libvirt.libvirt_lxc`, `community.libvirt.libvirt_qemu` | libvirt guests |
| `community.vmware.vmware_tools` | VMware Tools |

`ansible.builtin.paramiko_ssh` was removed in core 2.21, and the `smart` transport value was removed in core 2.20. Use `ssh`.

### Become plugins

| Plugin | Description |
| --- | --- |
| `ansible.builtin.sudo` | sudo; the default |
| `ansible.builtin.su` | su |
| `ansible.builtin.runas` | Windows Run As |
| `community.general.doas` | doas |
| `community.general.run0` | systemd run0 |
| `community.general.machinectl` | systemd machinectl |
| `community.general.pbrun`, `community.general.pmrun` | PowerBroker |
| `community.general.dzdo` | Centrify dzdo |
| `community.general.ksu` | Kerberos substitute user |
| `community.general.pfexec` | Solaris profiles |
| `community.general.sesu` | CA Privileged Access Manager |
| `community.general.sudosu` | sudo, then su |
| `ansible.netcommon.enable` | Network device enable mode |

### Callback plugins

| Plugin | Description |
| --- | --- |
| `default` | Standard output |
| `minimal` | Short output |
| `oneline` | One line per result |
| `tree` | Save results to a directory |
| `junit` | Write JUnit XML |
| `ansible.posix.profile_tasks` | Show task timing |
| `ansible.posix.profile_roles` | Show role timing |
| `ansible.posix.timer` | Show total run time |
| `ansible.posix.json` | Print JSON |
| `ansible.posix.jsonl` | Print JSON lines |
| `community.general.log_plays` | Log results to per-host files |
| `community.general.mail` | Send results by email |

```ini
[defaults]
stdout_callback = default
callback_result_format = yaml
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer
```

The old `community.general.yaml` callback was removed. Use the default callback with `callback_result_format = yaml`.

### Cache plugins

| Plugin | Description |
| --- | --- |
| `memory` | In memory for one run; the default |
| `jsonfile` | JSON files |
| `community.general.yaml` | YAML files |
| `community.general.pickle` | Pickle files |
| `community.general.redis` | Redis |
| `community.general.memcached` | Memcached |
| `community.mongodb.mongodb` | MongoDB |

### Custom filter example

```python
# filter_plugins/my_filters.py
def shout(value):
    return str(value).upper() + "!"


class FilterModule:
    def filters(self):
        return {"shout": shout}
```

```yaml
- name: Use a custom filter
  ansible.builtin.debug:
    msg: "{{ 'hello' | shout }}"
```

## Configuration

Ansible uses the first config file it finds, and settings are not merged. Check which one is active with `ansible --version`.

The search order is:

1. The `ANSIBLE_CONFIG` environment variable
2. `ansible.cfg` in the current directory
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

For any one setting, command line options override environment variables, and environment variables override the config file. Playbook keywords and connection variables override all of these, except extra vars keep top priority.

Ansible ignores `ansible.cfg` in a world-writable current directory.

### Sample ansible.cfg

```ini
[defaults]
inventory = ./inventories/prod/hosts.yml
remote_user = ansible
roles_path = ./roles
collections_path = ./collections
forks = 20
timeout = 30
host_key_checking = true
gathering = smart
fact_caching = jsonfile
fact_caching_connection = .cache/facts
fact_caching_timeout = 7200
stdout_callback = default
callback_result_format = yaml
callbacks_enabled = ansible.posix.profile_tasks
log_path = ./ansible.log
retry_files_enabled = false
interpreter_python = auto_silent
vault_password_file = ~/.vault_pass

[privilege_escalation]
become = false
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = true
retries = 3

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml

[diff]
always = false
context = 3

[galaxy]
server = https://galaxy.ansible.com
```

Config sections include `defaults`, `privilege_escalation`, `ssh_connection`, `persistent_connection`, `connection`, `colors`, `selinux`, `diff`, `galaxy`, `inventory`, `netconf_connection`, `jinja2` and `tags`. Core has 216 base settings, and `ansible-config list` documents them all.

### Common settings and environment variables

| `ansible.cfg` key | Environment variable | Default | Description |
| --- | --- | --- | --- |
| `home` | `ANSIBLE_HOME` | `~/.ansible` | Base directory |
| `inventory` | `ANSIBLE_INVENTORY` | `/etc/ansible/hosts` | Inventory source |
| `remote_user` | `ANSIBLE_REMOTE_USER` | Current user | Login user |
| `forks` | `ANSIBLE_FORKS` | `5` | Parallel processes |
| `timeout` | `ANSIBLE_TIMEOUT` | `10` | Connection timeout in seconds |
| `task_timeout` | `ANSIBLE_TASK_TIMEOUT` | `0` | Task timeout; 0 disables it |
| `transport` | `ANSIBLE_TRANSPORT` | `ssh` | Default connection plugin |
| `roles_path` | `ANSIBLE_ROLES_PATH` | `~/.ansible/roles`, then system paths | Role search path |
| `collections_path` | `ANSIBLE_COLLECTIONS_PATH` | `~/.ansible/collections`, then system path | Collection search path |
| `library` | `ANSIBLE_LIBRARY` | `~/.ansible/plugins/modules`, then system path | Module search path |
| `host_key_checking` | `ANSIBLE_HOST_KEY_CHECKING` | `true` | Check SSH host keys |
| `pipelining` | `ANSIBLE_PIPELINING` | `false` | Fewer SSH operations; needs no `requiretty` |
| `interpreter_python` | `ANSIBLE_PYTHON_INTERPRETER` | `auto` | Python discovery on targets |
| `gathering` | `ANSIBLE_GATHERING` | `implicit` | `implicit`, `explicit` or `smart` |
| `fact_caching` | `ANSIBLE_CACHE_PLUGIN` | `memory` | Fact cache plugin |
| `fact_caching_connection` | `ANSIBLE_CACHE_PLUGIN_CONNECTION` | None | Cache path or address |
| `fact_caching_timeout` | `ANSIBLE_CACHE_PLUGIN_TIMEOUT` | `86400` | Cache lifetime in seconds |
| `inject_facts_as_vars` | `ANSIBLE_INJECT_FACT_VARS` | `true` | Inject facts as top-level variables |
| `strategy` | `ANSIBLE_STRATEGY` | `linear` | Default strategy |
| `any_errors_fatal` | `ANSIBLE_ANY_ERRORS_FATAL` | `false` | Stop all hosts on any failure |
| `hash_behaviour` | `ANSIBLE_HASH_BEHAVIOUR` | `replace` | Replace or merge dictionaries |
| `stdout_callback` | `ANSIBLE_STDOUT_CALLBACK` | `default` | Output plugin |
| `callbacks_enabled` | `ANSIBLE_CALLBACKS_ENABLED` | Empty | Extra callbacks |
| `bin_ansible_callbacks` | `ANSIBLE_LOAD_CALLBACK_PLUGINS` | `false` | Load callbacks for ad hoc runs |
| `log_path` | `ANSIBLE_LOG_PATH` | None | Log file |
| `no_log` | `ANSIBLE_NO_LOG` | `false` | Hide task data in all tasks |
| `retry_files_enabled` | `ANSIBLE_RETRY_FILES_ENABLED` | `false` | Write `.retry` files |
| `vault_password_file` | `ANSIBLE_VAULT_PASSWORD_FILE` | None | Vault password file |
| `vault_identity_list` | `ANSIBLE_VAULT_IDENTITY_LIST` | Empty | Default vault IDs |
| `keep_remote_files` | `ANSIBLE_KEEP_REMOTE_FILES` | `false` | Keep module files on targets |
| `debug` | `ANSIBLE_DEBUG` | `false` | Print internal debug output |
| `enable_task_debugger` | `ANSIBLE_ENABLE_TASK_DEBUGGER` | `false` | Open the debugger on failure |
| `force_color` | `ANSIBLE_FORCE_COLOR` | `false` | Force colored output |
| `nocows` | `ANSIBLE_NOCOWS` | `false` | Disable cowsay output |
| `deprecation_warnings` | `ANSIBLE_DEPRECATION_WARNINGS` | `true` | Show deprecations |
| `display_skipped_hosts` | `ANSIBLE_DISPLAY_SKIPPED_HOSTS` | `true` | Show skipped hosts |
| `display_args_to_stdout` | `ANSIBLE_DISPLAY_ARGS_TO_STDOUT` | `false` | Show task arguments in headers |
| `editor` | `ANSIBLE_EDITOR`, `EDITOR` | `vi` | Editor for Vault |
| `pager` | `ANSIBLE_PAGER`, `PAGER` | `less` | Pager for documentation |
| `become` | `ANSIBLE_BECOME` | `false` | Enable become |
| `become_method` | `ANSIBLE_BECOME_METHOD` | `sudo` | Become method |
| `become_user` | `ANSIBLE_BECOME_USER` | `root` | Become user |

### Connection examples

```yaml
# group_vars/linux.yml
ansible_connection: ssh
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
ansible_ssh_common_args: -o ProxyJump=bastion.example.com
ansible_python_interpreter: auto_silent
```

```yaml
# group_vars/windows_winrm.yml
ansible_connection: winrm
ansible_port: 5986
ansible_winrm_scheme: https
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: validate
ansible_user: administrator
ansible_password: "{{ vault_win_password }}"
```

```yaml
# group_vars/windows_psrp.yml
ansible_connection: psrp
ansible_port: 5986
ansible_psrp_auth: ntlm
ansible_user: administrator
ansible_password: "{{ vault_win_password }}"
```

```yaml
# group_vars/windows_ssh.yml
ansible_connection: ssh
ansible_shell_type: powershell
```

```yaml
# host_vars/localhost.yml
ansible_connection: local
ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

### Privilege escalation

```yaml
- name: Become at play level
  hosts: all
  become: true
  become_user: root
  become_method: ansible.builtin.sudo
  tasks:
    - name: Run as another user
      ansible.builtin.command: whoami
      become: true
      become_user: postgres
      become_flags: -H -S
```

```bash
ansible-playbook site.yml -b -K                # Become, and prompt for the password
ansible-playbook site.yml -b --become-user app # Become a specific user
```

Switching to an unprivileged user needs extra care, because the remote user and the become user must share temporary files. Enable pipelining or use `setfacl` on the target.

## Debugging, testing and best practices

Check a change in stages: syntax, lint, dry run, one host, then all hosts.

### Safe rollout

```bash
ansible-playbook site.yml --syntax-check          # 1. Check syntax
ansible-lint site.yml                             # 2. Lint
ansible-playbook site.yml --list-hosts --list-tasks   # 3. Review the scope
ansible-playbook site.yml --check --diff          # 4. Dry run
ansible-playbook site.yml -l web01                # 5. Run on one host
ansible-playbook site.yml                         # 6. Run everywhere
```

### Debugging tools

```yaml
- name: Print a variable
  ansible.builtin.debug:
    var: my_var
    verbosity: 1           # Shows only with -v or higher

- name: Print a type
  ansible.builtin.debug:
    msg: "{{ my_var | type_debug }}"

- name: Show all host variables
  ansible.builtin.debug:
    var: hostvars[inventory_hostname]

- name: Check assumptions
  ansible.builtin.assert:
    that:
      - my_var is defined
      - my_var | int > 0
    fail_msg: my_var must be a positive number
    success_msg: my_var is valid

- name: Open the debugger when this task fails
  ansible.builtin.command: /bin/false
  debugger: on_failed
```

```bash
ANSIBLE_DEBUG=1 ansible-playbook site.yml              # Internal debug output
ANSIBLE_KEEP_REMOTE_FILES=1 ansible-playbook site.yml  # Keep module files on the target
ANSIBLE_LOG_PATH=./run.log ansible-playbook site.yml   # Log to a file
ANSIBLE_ENABLE_TASK_DEBUGGER=1 ansible-playbook site.yml   # Debug any failed task
ansible-playbook site.yml -vvvv                        # Show connection details
```

| Debugger command | Description |
| --- | --- |
| `p task` | Print the task |
| `p task.args` | Print task arguments |
| `p task_vars` | Print task variables |
| `p host` | Print the host |
| `p result._result` | Print the result |
| `task.args['key'] = 'value'` | Change an argument |
| `task_vars['key'] = 'value'` | Change a variable |
| `u`, `update_task` | Rebuild the task from changed values |
| `r`, `redo` | Run the task again |
| `c`, `continue` | Continue |
| `q`, `quit` | Quit the debugger |

### Exit codes

| Code | Meaning |
| --- | --- |
| `0` | Success |
| `1` | Error |
| `2` | One or more hosts failed |
| `3` | One or more hosts unreachable |
| `4` | Parser error |
| `5` | Bad or incomplete options |
| `99` | Interrupted by the user |
| `250` | Unexpected error |

### Common errors

| Error | Likely cause and fix |
| --- | --- |
| `UNREACHABLE` | Check SSH with `ssh -vvv`, the user, the key and `ansible_host` |
| `Permission denied` | Wrong user or key, or `become` is needed |
| `Missing sudo password` | Add `-K`, or set `ansible_become_password` from Vault |
| `Host key verification failed` | Add the key to `known_hosts`; avoid disabling checks |
| `The module ... was not found` | Install the collection and use the fully qualified name |
| `undefined variable` | Check the name and scope, or use `default()` |
| `Conditional result was ... non-boolean` | Make `when` return a boolean, as in `my_var \| length > 0` |
| `Attempting to decrypt but no vault secrets found` | Add `-J` or `--vault-password-file` |
| `Could not find or access` | Check `files/`, `templates/` and relative paths |
| `Timeout waiting for privilege escalation prompt` | Raise `timeout`, and check the sudo configuration |
| `python not found` | Set `ansible_python_interpreter`, or use `raw` to install Python |

### Version changes to watch

| Core version | Change |
| --- | --- |
| 2.19 | Stricter templating; conditionals must return booleans, and trusted templates are enforced |
| 2.20 | The `smart` transport value was removed; injected facts such as `ansible_distribution` were deprecated |
| 2.21 | `paramiko_ssh` was removed |
| 2.22 | Requires Python 3.13 or later on the control node; `ansible-config validate` also checks plugins by default |
| 2.24 | Planned: `INJECT_FACTS_AS_VARS` defaults to `False` |

### Best practices

- Use fully qualified names, such as `ansible.builtin.copy`.
- Name every task and play.
- Prefer a dedicated module to `command` or `shell`.
- Keep tasks idempotent; use `creates`, `removes` and `changed_when` with commands.
- Pin collection and role versions in `requirements.yml`.
- Store secrets in Vault or a secret manager, and set `no_log: true` on tasks that handle them.
- Keep separate inventories per environment.
- Keep playbooks short, and move logic into roles.
- Put overridable values in role `defaults`, and prefix them with the role name.
- Use `ansible_facts['name']`, not injected fact variables.
- Set `serial` and `max_fail_percentage` for production rollouts.
- Always set `mode` as a quoted string, such as `"0644"`.
- Run `ansible-lint` and `--check --diff` in CI.
- Test roles with `molecule`.

## Sources

Command options, plugin lists, config defaults and the module list were read from the installed `ansible` 14.5.0, `ansible-core` 2.21.6 and `ansible-core` 2.22.0 tools with `--help`, `ansible-doc` and `ansible-config list`.

- [Special variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)
- [Ansible 14 roadmap](https://docs.ansible.com/ansible/devel/roadmap/COLLECTIONS_14.html)
- [Porting guide for ansible-core 2.20](https://github.com/ansible/ansible-documentation/pull/3164/files)
- [Porting guide for Ansible 13](https://github.com/ansible/ansible-documentation/pull/3219/files)
- [The Bullhorn 219: paramiko\_ssh removal](https://forum.ansible.com/t/the-bullhorn-219/45456)
- [ansible-core release support](https://endoflife.date/ansible-core%20%20)
