# Testing FiveM resources end to end with real game clients

A field report, 1 November 2026.

We built a tool that lets an AI coding agent test FiveM resources the way a player hits them: two real game clients on one PC, real network events, hostile input, sync checks and the real in-game UI. No human touched the game during these runs. This report covers what worked, what didn't, and the things we measured that may save other FiveM developers some time.

The tool itself is private. Everything here was measured on a local test server, never a live one.

## Results

| Check | Result |
|---|---|
| Two clients launched, logged in on separate characters, spawned with no human input | **Works** |
| Fuzz checks against a deliberately broken resource | **10 of 10** correct |
| Test runner against the same resource | **10 of 10** correct: 5 real tests passed, and all 5 tests built to fail did fail, each for the right reason |
| Unit tests on FiveM's real Lua VM, with stubbed natives | **16 of 16** correct: 15 passed, and the 1 built to fail did |
| Real PvP resource, two-player suite | **9 of 9**, including a real gunshot and a real kill as controls |
| Real character screen, driven through its own buttons | **11 of 11** |
| Live UI on both clients, game-world screenshots, callbacks, client error logs | **20 of 20** |
| Every safety refusal in the tool | **26 of 26** |
| Forged forwarding headers against the loopback check | **7 of 7** refused |
| Three existing resource self-tests | **48 checks passed, 0 failed** |

## Bugs it found in shipped code

All three were in our shared bridge resource, were already live, passed that resource's own self-tests, and are now fixed.

1. **An unload event fired twice on every logout.** The bridge fired `QBCore:Client:OnPlayerUnload`, and then the framework's own `Logout` fired it again. 24 resources on the server listen for it, so each ran its unload logic twice. Measured 2 before the fix, 1 after.
2. **A documented export could never work.** A function passed through a FiveM export arrives as a callable table, not a function, so `type(fn) == 'function'` rejected every other resource. If your export takes a callback, check for `__cfx_functionReference` as well.
3. **Callbacks broke after a restart.** `lib.callback.register` adds a net handler and can't remove one. Registering again after the owning resource restarted left the dead handler in place, it answered first, and the caller got nothing back. Use one stable dispatcher per name.

Number 3 was hidden behind number 2, which is a good argument for testing the fix as well as the bug.

## Checks that lied to us

This is the most useful part of the report. Each of these gave a clean-looking result that was wrong.

1. **One input blinded the rest.** Sending `1e308` pushed a balance to infinity, and nothing could change it after that. A wide-open handler then reported "OK" 23 times. A fuzzer needs to rerun its normal case at the end and notice when state has stopped moving.
2. **A "no change" check that could never fail.** Reading the "before" value after firing the event just records the changed value as the baseline. Make the action an argument, so the order can't be wrong.
3. **A test that ran nothing looked like a pass.** Console-only commands fired from a script refuse silently: 0 passed and 0 failed. Zero checks should never read as success.
4. **Unfinished spawn looked like desync.** Character loaded isn't the same as in the world. Clients sitting on a spawn selector see each other invisible at `z = -190`, and every sync check fails for a reason unrelated to the code under test.
5. **Inspecting the wrong client.** The DevTools port follows launch order, not which client you think it is. Have each client announce itself.
6. **A native that doesn't exist got through.** Aliases starting with `_` in the natives list aren't callable under the plain name. `SetPedHairColor` reads like a real native and throws in game.
7. **`ok and x or y`.** A step returning `false` came back as the string `"false"`, because Lua falls through to `y` when `x` is false.
8. **A restart reported success with half the server down.** Predicting dependents from manifests missed real ones. Compare what was running before and after instead.

## Two wrong diagnoses

We blamed number 1 above on JSON turning integers into floats. The next check disproved it. It sounded completely reasonable, which is the point.

We also spent a long time on a gunshot test that fired but did no damage, and blamed the spawn bug. That explained the sync failures only. Then the person watching the game said the gun really was firing, so we measured where the bullets landed: **`TaskShootAtEntity` put them 38 m from a target standing 2 m away.** A bullet with a computed path from `ShootSingleBulletBetweenCoords` landed first time, and the whole hit pipeline worked. A real observation beat several rounds of reasoning.

## What is still limited

- **The rejection tests only mean something next to a control.** Forged hit reports were refused all along, but until a real hit landed, a broken pipeline would have passed too. Write the control first.
- **184 hostile inputs against the PvP events showed no crashes and no money changes.** The normal arguments don't move those values either, so this says less than the number suggests. The two-player suite is what proves the hit logic.
- **Only Qbox was run.** The fixes were read against QBCore's source, not run on it. ESX wasn't tested.
- **Flood numbers are four runs on one PC.**
- **The test server's checkout can trail the live server,** so a test here isn't automatically proof about production.

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

- About 4.5 GB RAM per client and 0.5 GB for the server. VRAM reached 3.8 of 4 GB.
- Free RAM once dropped to 0.5 GB with a browser open.
- Qbox's `checkDuplicateLicense` has to be off on the test server. Both clients share one license, so they share one character list.
- Logging into a character already in use gets you kicked by ox_inventory.
- ox_inventory strips any weapon the player doesn't own, so a test pistol must be a real item, and it starts unloaded.
- `TaskShootAtEntity` and `TaskShootAtCoord` are unreliable for tests. A computed `ShootSingleBulletBetweenCoords` from 1 m in front of the target works.

### Tools

- Playwright's `connectOverCDP` hung under Bun and worked under Node. All resource UIs show up as frames inside `nui://game/ui/root.html`.
- FiveM's real Lua VM is public: `citizenfx/lua`, branch `luaglm-548`. It built with MinGW's `make mingw`, and `code/vendor/lua.lua` in `citizenfx/fivem` lists the flags FiveM uses. `+=`, `?.`, backtick hashes, `vector3`, set constructors, `local a, b in t` and `/* */` all work. `//` does not start a comment, because it's integer division.
- `screencapture` on the server (it provides `screenshot-basic`) returned a real PNG of the rendered game. DevTools screenshots only show the UI layer.
- Stock `luac5.4 -p` rejects valid CfxLua such as `+=`. Check syntax with the real compiler.
- `ExecuteCommand('ensure x')` needs ACEs for both stop and start, and a missing one only prints "Access denied". `StopResource` and `StartResource` don't need them.
- `SetHttpHandler` runs on the game port, so "bind it to localhost" isn't an option in the handler itself.

### Network events

Four runs on one PC, so treat them as ballpark.

| Events sent | Per tick | Arrived | Result |
|---|---|---|---|
| 100 | 1 | 100 | Fine |
| 500 | 5 | 241 to 268 | About half lost, client stayed connected |
| 2000 | 20 | 123 to 136 | Client kicked: "Reliable network event overflow" |

Through msgpack, a trailing nil argument is dropped, so the argument count falls. A nil in the middle stays as a hole. NaN, inf and -inf all arrive intact.

## Credits

The RCON client and log lookup are adapted from [ziyacivan/fivem-mcp](https://github.com/ziyacivan/fivem-mcp), and the DevCon frame format came from reading it (MIT). The Lua VM is [citizenfx/lua](https://github.com/citizenfx/lua). Built with Claude Code.
