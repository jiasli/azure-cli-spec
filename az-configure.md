# A more versatile `az configure`

## Background knowledge

In `git`, when [setting username](https://help.github.com/en/github/using-git/setting-your-username-in-git) and [email](https://help.github.com/en/github/setting-up-and-managing-your-github-user-account/setting-your-commit-email-address), the commands are

```
git config --global user.name "Mona Lisa"
git config --global user.email "email@example.com"
```

The corresponding `~/.gitconfig`:

```ini
[user]
	name = Mona Lisa
	email = email@example.com
```

`az configure` currently only supports setting and listing the `default` section:

```
> az configure -h

Command
    az configure : Manage Azure CLI configuration. This command is interactive.

Arguments
    --defaults -d      : Space-separated 'name=value' pairs for common argument defaults.
    --list-defaults -l : List all applicable defaults.  Allowed values: false, true.
    --scope            : Scope of defaults. Using "local" for settings only effective under current
                         folder.  Allowed values: global, local.  Default: global.
```

For other sections described in [Azure CLI configuration](https://docs.microsoft.com/en-us/cli/azure/azure-cli-configuration?view=azure-cli-latest), users will have to manually edit the `~/.azure/config` file, like 

```ini
[core]
collect_telemetry = no
no_color = yes

[logging]
enable_log_file = no
```

This is inconvenient. To improve the user experience, `az configure` needs to ...

## Support set, get and unset

These can be implemented in 2 ways:

### `git config` convention

```
az configure --set ...
az configure --get ...
az configure --unset ...
```

### Azure CLI convention - sub-commands

`az configure` is already used as a command. It is not possible to use it as a command group anymore. We need a another command group, like `config`:

```
az config set ...
az config show ...
az config remove ...
```

## Support section

However, as Azure CLI doesn't support positional arguments, the config entry has to be prefixed by a parameter name like `--name`.

Also there is a design decision for how to divide the config entry:

```
[section].[name]=[value]
```

### Don't extract anything

```sh
az configure --set core.no_color=true  # personally I prefer this format
# or
az config set --name core.no_color=true
```

### Extract `--section`

```sh
az configure --section core --set no_color=true
# or
az config --set --section core --name no_color=true
```

### Extract `--value`

```sh
az configure --set core.no_color --value true
# or
az config set --name core.no_color --value true
```

### Extract `--section` and `--value`

```sh
az configure --section core --set no_color --value true
# or
az config set --section core --name no_color --value true
```

⚠ Anyway, it is the destiny of `--defaults` and `--list-defaults` to be deprecated.

## Benefits

This will greatly simplify the configuration process and make experimental features easier to use, like 

```
az configure --set core.no_color=true
az configure --set core.command_index=on
az configure --set logging.log_dir=D:\logs
```

## Additional work - migrate to settings.json

`settings.json` is the latest unwritten convention of Microsoft open source project, like [Visual Studio Code](https://github.com/microsoft/vscode) and [Windows Terminal](https://github.com/microsoft/terminal).

### Visual Studio Code

The config file is at `C:\Users\xxx\AppData\Roaming\Code\User\settings.json`:

```jsonc
{
    "terminal.integrated.shell.windows": "C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
    "terminal.integrated.scrollback": 99999,
    "editor.minimap.enabled": false,
    ...
}
```

### Windows Terminal

The config file is at `C:\Users\xxx\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json`:

```jsonc
{
    "$schema": "https://aka.ms/terminal-profiles-schema",

    "defaultProfile": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",

    "profiles":
    [
        {
            // Make changes here to the powershell.exe profile
            "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
            "name": "Windows PowerShell",
            "commandline": "powershell.exe",
            "hidden": false
        },
        ...
    ],
    ...
}
```

Using `settings.json` will open up more possibilities like "hover" help (controlled by `editor.hover.enabled` in VSCode) using the `"$schema"` property:

![image](https://user-images.githubusercontent.com/4003950/84798582-b8488a00-b02d-11ea-9d88-935398c4d60f.png)
