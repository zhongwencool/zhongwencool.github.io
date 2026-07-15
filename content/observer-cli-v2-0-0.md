---
title: "From reading code to seeing the runtime: observer_cli 2.0 is out"
date: 2026-07-14
tags:
  - observer_cli
  - Erlang
  - Elixir
  - BEAM
  - AI
---

Your AI assistant can read an entire Erlang or Elixir project. It can trace a call path, add tests, and sometimes fix a bug that has been hiding for months.

Then you ask:

> Why did the production node suddenly slow down?

And it reaches the edge of the repository.

Source code tells the assistant how the system should behave. The live BEAM node knows what is happening now: which process mailboxes are growing, where memory went, whether schedulers are busy, and which connections are in trouble. Until now, a person usually had to open the observer_cli TUI, inspect those views, and relay the findings back to the AI.

[observer_cli v2.0.0](https://github.com/zhongwencool/observer_cli/releases/tag/v2.0.0) changes that handoff.

## First, what is observer_cli?

observer_cli is a terminal diagnostic tool for live Erlang and Elixir systems. It connects over Erlang distribution and inspects processes, memory, schedulers, ETS, Mnesia, ports, sockets, and applications.

It has traditionally worked best with a person sitting in front of its TUI. Version 2.0 keeps that interface and adds a complete non-interactive CLI. An AI assistant that can run shell commands can now gather runtime evidence itself instead of waiting for screenshots or pasted output.

## Skip the installation tutorial. Send the docs

Give your AI assistant the machine-readable CLI guide:

**https://hexdocs.pm/observer_cli/cli.md**

Then say:

```text
Follow this guide to configure observer_cli 2.0 in the current project and inspect my target node.
Preserve unrelated changes. Ask me if the node or cookie source is missing.
Do not deploy, restart the node, or enable tracing without my approval.
When finished, tell me what changed, what you found, and which checks did not complete.
```

It can handle the mechanical work: detect Mix or Rebar3, prepare the target release, select the command-line build for the local OTP version, then use `connect` and `status` to verify the target before starting the diagnosis.

You still control the decisions that matter: the target node, cookie source, deployment window, and whether a more invasive operation such as tracing is allowed.

## What an investigation looks like now

Suppose a production service starts showing higher latency. Instead of opening the TUI, searching through several views, and pasting the results into a chat, you can ask:

> Check for capacity pressure first. Follow the evidence before drilling down, and report any checks that did not complete.

The assistant starts with `diagnose` for triage, then uses the evidence to choose a narrower follow-up command.

The important part is not that the AI can run more commands. It is that every command tells it what evidence was actually collected.

Structured responses in observer_cli 2.0 report whether collection completed, which probes ran, what they found, and what failed. “No problem found” no longer has to mean “the check never finished.”

### The same alert can lead to several investigation paths

Think of `diagnose` as triage, not the only answer. Its real value is giving the assistant enough evidence to choose the next command.

**Requests are getting slower.** The assistant can use `schedulers` to check scheduler pressure, then rank `processes` by mailbox size, memory, or reductions. Once it finds a suspicious PID, `process` returns bounded metadata and the current stack. If the process is known to be a `gen_server`, `gen_statem`, or `gen_event`, `otp-state` can inspect a constrained shape of its state.

**Memory keeps growing.** `memory` shows the overall BEAM memory and allocator picture. `applications` groups process resources by application. If the evidence points to tables, `ets` and `mnesia` expose table metadata without reading entire table contents.

**Nodes or connections are unstable.** `distribution` shows connected Erlang nodes, `network` measures VM network I/O, and `sockets` inspects the OTP socket registry. If the problem involves Erlang Port objects or driver queues, the assistant can continue with `ports` and `port`.

**You need to preserve the scene.** `snapshot` captures a bounded point-in-time VM bundle. `logs` reads a limited retained tail from a supported Logger file. If one critical gap remains, the assistant can recommend an exact `trace call`; you still decide whether it runs.

When an application keeps restarting, `supervision-tree` can add the application root supervisor and its direct children to the picture. The goal is not to run every command. It is to start with the symptom and follow the evidence.

## Built for production, not a demo

When an AI assistant can reach a production node, the main risk is not that it will do too little. It is that it will do too much.

Version 2.0 therefore puts count, duration, or size limits around lists, sampling, logs, and tracing. Every remote command starts a temporary local controller process and exits when it is done. `connect` remembers where to obtain the cookie, not the cookie value. The command interface also refuses to inject missing diagnostic code into the target for convenience.

The higher-risk `trace call` command requires an exact PID and module/function/arity (MFA), plus explicit acknowledgement. The assistant can recommend that step; you still decide whether it runs.

These constraints are less dramatic than a promise to “fix everything automatically,” but they are closer to what production diagnostics actually needs.

## The TUI is still here

The TUI remains the better interface when you want to watch values change, change rankings, or drill through processes yourself. Version 2.0 simply adds a second path: people can keep exploring, while automation and AI assistants finally get a stable entry point.

This release also introduces the explicit `tui` command and updates the TUI plugin API, runtime views, and release flow. See the complete [Changelog](https://github.com/zhongwencool/observer_cli/blob/v2.0.0/docs/CHANGELOG.md).

## The complete command surface

Below is the `observer_cli --help` output from v2.0.0. Readers do not need to memorize it. An AI assistant can discover the available capabilities here and use `COMMAND --help` for exact limits and examples.

```bash
observer_cli --help
Usage:
  observer_cli COMMAND [ARGUMENTS] [OPTIONS]
  observer_cli tui NODE [COOKIE REFRESH_MS]
  observer_cli connect --node NODE
    (--cookie-env NAME | --cookie-file PATH)

Target context:
  connect             Verify and save a target context
  status              Check the saved target context
  disconnect          Remove the saved target context

Diagnostics:
  diagnose            Detect likely VM problems and report evidence
  snapshot            Collect a point-in-time VM fact bundle
  logs                Read bounded retained logs from one configured file

VM health:
  memory              Show VM memory and allocator usage
  schedulers          Measure scheduler utilization and run queues
  distribution        Show connected Erlang nodes
  network             Show VM network I/O

Runtime resources:
  processes           List top processes
  process PID_OR_NAME Inspect one process
  applications        Group process resources by application
  ets                 List ETS tables
  mnesia              List local Mnesia tables
  ports               List Erlang ports
  port PORT_ID        Inspect one Erlang port
  sockets             List OTP sockets

OTP structures:
  otp-state PID_OR_NAME
                      Inspect an asserted OTP behavior state shape
  supervision-tree --app APP
                      Show an application supervision tree

Tracing:
  trace call MFA      Run a bounded function trace
  trace stop --all    Clear all node-static tracing

Interactive:
  tui NODE            Open the terminal UI; see 'tui --help'

Help and version:
  --help, -h          Show this overview
  help COMMAND        Show command help (also COMMAND --help)
  --version           Show local bundle, protocol, schema, and OTP

Target options:
  --node NODE         Use an explicit target instead of saved context
  --cookie-env NAME   Read the target cookie from an environment variable
  --cookie-file PATH  Read the target cookie from a file
  --name-mode MODE    short or long; inferred from NODE by default

Identifier policy:
  snapshot and diagnose redact identifiers by default.
  Inspection and tracing include identifiers by default.

Output options:
  --format FORMAT     text, term, or json; text by default
  --json              Alias for --format json (OTP 27+ controller)
  --redact            Hide target identifiers
  --include-identifiers
                      Include identifiers in snapshot or diagnose output
  --timeout DURATION  Set the command deadline, up to 120s

DURATION accepts milliseconds (1500 or 1500ms) or seconds (2s).

Exit status: 0 success, 1 diagnose findings, 2 usage/capability,
3 runtime/refusal/partial, 4 internal/schema/cleanup.

Run 'observer_cli COMMAND --help' for options and examples.

```

The next time you ask an AI assistant what is happening inside a BEAM node, do not start with a terminal screenshot.

Send it the [observer_cli 2.0 CLI guide](https://hexdocs.pm/observer_cli/cli.md).
