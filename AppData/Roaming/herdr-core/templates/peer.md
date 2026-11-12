---
description: Find yourself and other agents in this Herdr space, and talk to them with hc.
command: pi
---
This is only how to find agents and talk to them. It does not decide the task. The person's message is the task.

## Where you are

Your address is `$HC_TERMINAL_ID`.

```sh
hc get "$HC_TERMINAL_ID" --json
```

`labels.herdr.space` is the workspace. If it is missing, say so and stop. Do not use `cwd` as the workspace. In `hc ls`, `cwd` is the launch directory, not the current directory.

## Where the others are

```sh
hc ls --label "herdr.space=<that value>" --json
```

Remove your own address. Every remaining row is another agent in this workspace. Use its `address` exactly as listed, including a `machine/` prefix.

If more than one remains and the person did not name one, show the addresses and ask. Do not message all of them.

## Send

```sh
hc send <address> "peer: $HC_TERMINAL_ID <text>"
```

`hc send` pastes that text and presses Enter. You do not need to have started the other terminal.

- Begin with `peer:` and your address, so the receiver can reply without searching.
- Keep the send short. Put long text in a workspace file and include its path.
- Do not use `hc send --force`.
- Do not expect an `<hc from="...">` tag. That tag is only a child reporting to the terminal that started it.
- `hc wait --children` does not include these agents. A send arrives when the other agent next reads input. Do not sleep or poll.

A message that starts with `peer:` is another agent, not a new task from the person. Reply with `hc send` to the address written in it.
