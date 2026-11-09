# Stop-path census of robot MCP servers (v0.1)

When an AI agent moves a robot through an MCP server and someone asks it to stop while the move is still running, does the stop get through? This census answers that from source code alone. Each server was read at a pinned commit, and every line cited here was read from the raw file at that commit.

`census.csv` has 15 servers. Each row gives the full commit SHA, a class, the lines that make the motion last and the lines that make up the stop (file:line plus a verbatim quote of at most 100 characters), one sentence of reasoning, and whether a second reader has checked it.

To open any cited file, use `https://raw.githubusercontent.com/<repo>/<commit>/<path>`. Line numbers count from 1 in that raw file.

## The classes, in plain words

| class | meaning |
|---|---|
| **A** blocking | The motion tool keeps the server busy for the whole move: it sleeps, loops or waits on the move inside the server's only event loop. A stop sent during the move is handled only after the move ends. |
| **B** no stop | The server offers no tool that stops, halts or cancels motion. |
| **C** own-goals stop | A stop exists, but it cancels only the motions this server started (by its own handle or ID). |
| **D** preemptible | The motion runs on the device, in a worker thread or in a cancellable task, and a stop is handled while the motion runs. |
| **U** unknown | The path from the tool to the robot cannot be seen in the code. |

A server can carry two letters (A+B, C+D).

v0.1 tally: A+B 2, A 2, B 4, C+D 2, D 4, U 1.

## Framework rules the classes rely on

Whether a stop is handled during a move depends on how the MCP framework dispatches requests. These are the rules used, each with its source.

1. **Python MCP SDK 1.x, low-level server: each request runs in its own task on one event loop.** python-sdk tag `v1.27.0`, `src/mcp/server/lowlevel/server.py`: `async with anyio.create_task_group() as tg:` · `async for message in session.incoming_messages:` · `tg.start_soon(` (lines 673, 675 and 678).
2. **Python MCP SDK 1.x, FastMCP: a plain `def` tool is called directly, on that loop.** Same tag, `src/mcp/server/fastmcp/utilities/func_metadata.py`: `if fn_is_async:` · `return await fn(**arguments_parsed_dict)` · `return fn(**arguments_parsed_dict)` (lines 92-95). `src/mcp/server/fastmcp/tools/base.py`: `result = await self.fn_metadata.call_fn_with_arg_validation(`. Neither file contains `to_thread`, `run_sync` or `threadpool`.
3. **Python asyncio: a blocking call inside a coroutine delays every other task on the loop.** Python docs, "Developing with asyncio", Running Blocking Code (https://docs.python.org/3/library/asyncio-dev.html): "For example, if a function performs a CPU-intensive calculation for 1 second, all concurrent asyncio Tasks and IO operations would be delayed by 1 second." The same holds for `time.sleep` or a blocking socket wait.
4. **TypeScript MCP SDK v1.27.1: requests are handled concurrently, and a client cancel reaches only handlers that use the abort signal.** typescript-sdk tag `v1.27.1`, `src/shared/protocol.ts` (exact lines): 628 `this._transport.onmessage = (message, extra) => {` · 633 `this._onrequest(message, extra);` · 685 `private _onrequest(request: JSONRPCRequest, extra?: MessageExtraInfo): void {` · 723 `const abortController = new AbortController();` · 780 `.then(() => handler(request, fullExtra))` · 550 `private async _oncancel(notification: CancelledNotification): Promise<void> {` · 556 `controller?.abort(notification.params.reason);` · 241 `signal: AbortSignal;`. `_onrequest` returns `void` and its promise chain is not awaited, so the next message is read at once.

The rows' own dependency pins are given where they were read (for example `"mcp[cli]>=1.6.0",`). Rules 1-2 were read at one tag, v1.27.0. The version a user's install resolves to was not read. Where a row has no upper pin below 2.0, its class assumes a 1.x SDK that imports `mcp.server.fastmcp` and behaves as rules 1-2 describe.

## Method

- **Static and read-only.** Nothing was installed, run or connected to. No robot and no simulator was used.
- **Pinned.** Each row names a full 40-character commit. A project may have changed since.
- **Verbatim.** Quotes are exact substrings of the cited line. Lines over 100 characters are quoted in part.
- **Scope.** The census covers the MCP server's own tools only. A robot's hardware emergency stop, a teach pendant, a vendor controller's stop or the operator's power switch is outside the server and outside this census. A class says nothing about those.
- **What a class is not.** A class describes one code path at one commit. It is not a test of a robot and not a rating of a project. Several of these servers are research or hobby code that never claimed to offer a stop.
- **Second reads.** The `second_read` column names who re-read a row from the raw files. "pending" means only the author has read it so far.

## Prior art

- Owotogbe et al., "A Taxonomy of Runtime Faults in Model Context Protocol Servers", arXiv 2606.05339 (v1 2026-06-03; a v2 is dated 2026-09-19; quoted here from v1). It covers 473 MCP server repositories, including "timeouts or explicit cancellations of in-progress operations". It does not focus on robots.
- Grove (an AI agent at ChatForest), "Robotics MCP Servers — ROS, Home Assistant, ESP32, Robot Arms, Drones, and More", ChatForest, https://chatforest.com/reviews/robotics-mcp-servers/ (published 2026-03-19, updated 2026-08-16), a review of robotics MCP servers. It was also posted on dev.to: https://dev.to/grove_chatforest/robotics-mcp-servers-ros-home-assistant-esp32-robot-arms-drones-and-more-3me5

We found no earlier census of how a stop reaches the robot during a running motion. If you know of one, please open an issue.

## Disclosure

BotInfo publishes BotInfo (https://github.com/botinfo-ai/botinfo), which its README calls "an open-source policy layer that sits between an AI agent and a robot its owner has connected", and the BotInfo Benchmark, which grades robot connectors. So we have an interest in this question. The census was written by an AI research agent at BotInfo.ai.

## Corrections

If a row is wrong, or a newer commit changes the stop path, please open an issue with the file and line. Rows are corrected against the raw file and re-pinned.
