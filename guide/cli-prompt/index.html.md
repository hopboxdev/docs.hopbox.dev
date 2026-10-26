# Work in the prompt of the CLI

> How to work in the prompt of the hopbox CLI: Tab completion of commands and live machine names, quotes, a shell, Ctrl-C, the history, and profiles.

This guide opens the prompt of the hopbox CLI and works in it. The prompt runs one command after another, and the Tab key completes commands, flags, and machine names. [hopbox CLI](/reference/cli/index.html.md#the-prompt) lists every key, rule, and exit status of the prompt.

## What you need

- The CLI, signed in to an install, as in [Use the hopbox CLI](/guide/cli/index.html.md).
- A terminal. The prompt does not open in a script, in a pipe, or with `--json`.
- For the machine names, an install with a node, as in [Use machines from the CLI](/guide/cli-machines/index.html.md). Without a node, the prompt still runs every command, and Tab completes the commands and the flags.

## Open the prompt

Run `hopbox` without a command:

```sh
hopbox
```

The prompt shows:

```text
hopbox>
```

Type a command without the word `hopbox`, and press Enter:

```text
hopbox> list
NAME  STATE    SIZE     SHARED  URL                       SSH
box1  running  default  no      https://box1.example.com  box1.hop
hopbox>
```

Each line runs as the same command does in a shell. The output, the errors, and the effects are the same. A line that starts with `hopbox` also works, so you can paste a command from a shell or from these docs.

## Complete with Tab

Type the start of a command, and press Tab:

```text
hopbox> mach<Tab>
hopbox> machine 
```

If many words fit, Tab shows them all:

```text
hopbox> machine s<Tab>
show     start    stop     share    sizes
```

Press Tab again to move through the list, and Enter to take a word.

After a command that names a machine, Tab completes the names of the machines of the org:

```text
hopbox> connect b<Tab>
hopbox> connect box1 
```

The names are live. In a second terminal, make a machine:

```sh
hopbox new box2
```

Back in the prompt, `connect box2` completes at once, with no new start of the prompt. For `cp`, Tab adds a `:` after the name, as in `box1:`, so you can type the path in the machine next.

A word that starts with `-` completes the flags. For a flag with a fixed set of values, the values complete too:

```text
hopbox> new box3 --restart <Tab>
always   never
```

## Use quotes as in a shell

The prompt splits a line into words as a shell does. Quotes keep a space in a word:

```text
hopbox> exec box1 -- sh -c 'echo "hello from $(hostname)"'
hello from box1
```

The prompt does no expansion other than `~` for your home dir. In the line above, the shell in the machine expands `$(hostname)`, not the prompt.

## Open a shell, then come back

`connect` opens a shell in the machine, as in a shell on your laptop:

```text
hopbox> connect box1
user@box1:~$ exit
hopbox>
```

The shell gets every key until it ends. Then the prompt gets the keys again.

## Stop a command

A Ctrl-C stops the command that runs, and the prompt stays open:

```text
hopbox> logs -f box1
hello
^Chopbox: canceled: context canceled
hopbox>
```

At the prompt itself, Ctrl-C drops the line that you typed, and nothing runs.

## Find an earlier line

Up and Down move through the lines that you ran. Ctrl-R searches them: press Ctrl-R, type a part of the line, and press Enter to run the line that it found. The prompt keeps the last 1000 lines in `~/.config/hopbox/history`, so they are there the next time too.

The history leaves out a line that can hold a secret, such as `env set TOKEN=value` and `org join`. To keep any other line out, start it with a space. To set a secret, leave the value out, and type it at the question, where it does not show on the screen:

```text
hopbox> env set TOKEN --secret --scope private
Value of TOKEN:
Set the secret TOKEN in the scope private.
```

## Use another profile

Global flags before no command apply to each line of the prompt. The prompt names a profile other than `default`:

```sh
hopbox --profile dev
```

```text
hopbox (dev)> whoami
```

`hopbox profile use` and `hopbox org use` in the prompt change the profile and the org of the next lines, as in a shell. The machine names then come from the new org.

## Close the prompt

Type `exit` or `quit`, or press Ctrl-D on an empty line. The CLI exits with status 0, and the terminal is as it was before.
