# Testing FiveM resources end to end with real game clients

A field report, 1 November 2026.

We built a tool that lets an AI coding agent test FiveM resources the way a player hits them: two real game clients on one PC, real network events, hostile input, sync checks and the real in-game UI. No human touched the game during these runs. This report covers what worked, what didn't, and the things we measured that may save other FiveM developers some time.

The tool itself is private. Everything here was measured on a local test server, never a live one.

## Results

| Check | Result |
|---|---|
| Two clients launched, logged in on separate characters, spawned with no human input | **Works** |
| Fuzz oracle against a deliberately broken resource | **10 of 10** checks correct |
| Test runner against the same resource | **10 of 10** correct: 5 real tests passed, and all 5 tests built to fail did fail, each for the right reason |
| Unit tests on FiveM's real Lua VM, with stubbed natives | **8 of 8** correct: 7 passed, and the 1 built to fail did |
| Two-player sync check | **Works**, including catching an injected state bag desync |
| Live UI inspection on both clients, plus offline UI tests | **Works**, and a changed screenshot fails the baseline diff |
| Three existing resource self-tests | **48 checks passed, 0 failed** |
| Real PvP resource, 7 two-player tests | **6 passed, 1 failed** (see below) |
| Real PvP resource, 184 hostile inputs | **No errors or money changes** |

## What is not proven

**The gunshot control test still fails.** A loaded pistol fires at a visible target 2 m away, ammo drops, and the coordinates match on both sides, but no damage lands and the server sees no damage event. We don't know why yet.

That matters. The tests showing forged hit reports are rejected all pass, but without a working real hit to compare against, they can't yet prove the hit pipeline works. A broken pipeline would pass them too.

**The 184 hostile inputs prove less than the number suggests.** The normal arguments didn't move any state we were watching either, so those runs show no crashes and no money changes. They don't show the handlers were doing anything.

Also not tested:

- A spoofed forwarding header against the HTTP loopback check.
- Whether character loaded events fire exactly once with two clients.
- Whether our Lua build flags match FiveM's own build.
- Screenshots of the game world. UI layer capture works, and it has no game behind it.

## Checks that lied to us

This is the most useful part of the report. Each of these gave a clean-looking result that was wrong.

1. **One input blinded the rest.** Sending `1e308` pushed a balance to infinity, and nothing could change it after that. A wide-open handler then reported "OK" 23 times. A fuzzer needs to rerun its normal case at the end and notice when state has stopped moving.
2. **A "no change" check that could never fail.** Reading the "before" value after firing the event just records the changed value as the baseline. Make the action an argument, so the order can't be wrong.
3. **A test that ran nothing looked like a pass.** Console-only commands fired from a script refuse silently: 0 passed and 0 failed. Zero checks should never read as success.
4. **Unfinished spawn looked like desync.** Character loaded isn't the same as in the world. Clients sitting on a spawn selector see each other invisible at `z = -190`, and every sync check fails for a reason unrelated to the code under test.
5. **Inspecting the wrong client.** The DevTools port follows launch order, not which client you think it is. Have each client announce itself.
6. **A native that doesn't exist got through.** Aliases starting with `_` in the natives list aren't callable under the plain name. `SetPedHairColor` reads like a real native and throws in game.

We also got one diagnosis wrong along the way. We blamed number 1 on JSON turning integers into floats, and the next check disproved it. Worth saying, because it sounded completely reasonable.

## Measured facts

Setup: Ryzen 7 5700X, 32 GB RAM, GTX 1650 Super 4 GB, FXServer `v1.0.0.35245`, game build 3258 (Legacy), Qbox with ox_lib and ox_inventory.

### Launching

- FiveM refuses to start from a script: "This application should be launched directly from the shell or a web browser."
- A Windows shortcut opened through `explorer.exe` works, with `+connect 127.0.0.1:30120` in its arguments. Add `-cl2` for the second client. This is the shortcut method from FiveM's docs. No anti-cheat changes are involved, and none should be.
- In-game in about a minute.

### Two clients on one PC

| Thing | First client | Second client |
|---|---|---|
| NUI DevTools | `127.0.0.1:13172` | `[::1]:13172` |
| DevCon | `29200` | `29300` |
| Log line marker | `FiveM_b` | `FiveM_cl2_` |

- About 4.5 GB RAM per client and 0.5 GB for the server. VRAM reached 3.2 of 4 GB.
- Free RAM once dropped to 0.5 GB with a browser open.
- Qbox's `checkDuplicateLicense` has to be off on the test server. Both clients share one license, so they share one character list.
- Logging into a character already in use gets you kicked by ox_inventory.
- ox_inventory strips any weapon the player doesn't own, so a test pistol must be a real item, and it starts unloaded.

### Tools

- Playwright's `connectOverCDP` hung under Bun and worked under Node. All resource UIs show up as frames inside `nui://game/ui/root.html`.
- FiveM's real Lua VM is public: `citizenfx/lua`, branch `luaglm-548`. It built with MinGW's `make mingw`. `+=`, `?.`, backtick hashes, `vector3`, set constructors, `local a, b in t` and `/* */` all work. `//` does not start a comment, because it's integer division.
- Stock `luac5.4 -p` rejects valid CfxLua such as `+=`. Check syntax with the real compiler.
- `ExecuteCommand('ensure x')` needs ACEs for both stop and start, and a missing one only prints "Access denied". `StopResource` and `StartResource` don't need them.
- `SetHttpHandler` runs on the game port, so "bind it to localhost" isn't an option in the handler itself.

### Network events

Single measurements, so treat them as ballpark.

| Events sent | Per tick | Arrived | Result |
|---|---|---|---|
| 100 | 1 | 100 | Fine |
| 500 | 5 | 250 | Half lost, client stayed connected |
| 2000 | 20 | 128 | Client kicked |

Through msgpack, a trailing nil argument is dropped, so the argument count falls. A nil in the middle stays as a hole. NaN, inf and -inf all arrive intact.

## Credits

The RCON client and log lookup are adapted from [ziyacivan/fivem-mcp](https://github.com/ziyacivan/fivem-mcp), and the DevCon frame format came from reading it (MIT). The Lua VM is [citizenfx/lua](https://github.com/citizenfx/lua). Built with Claude Code.
