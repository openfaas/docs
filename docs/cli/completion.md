# Shell Completion
The OpenFaas CLI has a shell completion feature for Bash and ZSH.

## Bash for Linux
Bash for Linux requires [bash-completion](https://github.com/scop/bash-completion), please follow their installation [here](https://github.com/scop/bash-completion#installation).

Verify if it was installed correctly by executing the following command `type _init_completion`.

You can enabled OpenFaaS completion with two different commands:
```bash
$ echo 'source <(faas-cli completion --shell bash)' >>~/.bashrc
```

Or:
```bash
$ faas-cli completion --shell bash > /etc/bash_completion.d/faas-cli
```

## Bash for MacOS
By default MacOS comes with Bash 3.2, but the completion requires version 4.1+. See this [instructions](https://itnext.io/upgrading-bash-on-macos-7138bd1066ba) to upgrade.

Also, you'll need bash-completion v2. Install with the following commands:
```bash
$ brew install bash-completion@2
```

And add the following to your `~/.bashrc` file:
```bash 
export BASH_COMPLETION_COMPAT_DIR="/usr/local/etc/bash_completion.d"
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
```

You can enabled OpenFaaS completion with two different commands:
```bash
$ echo 'source <(faas-cli completion --shell bash)' >>~/.bashrc
```

Or:
```bash
$ faas-cli completion --shell bash > /etc/bash_completion.d/faas-cli
```

## ZSH
ZSH completion is simpler, just execute the following command:

```bash
source <(faas-cli completion --shell zsh)
```