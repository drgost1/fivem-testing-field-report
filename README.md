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
| Unit tests on FiveM's real Lua VM, with stubbed natives | **17 of 17** correct: 16 passed, and the 1 built to fail did |
| Real PvP resource, two-player suite | **9 of 9**, including a real gunshot and a real kill as controls |
| Real character screen, driven through its own buttons | **11 of 11** |
| Live UI on both clients, game-world screenshots, callbacks, client error logs | **20 of 20** |
| Every safety refusal in the tool | **26 of 26** |
| Forged forwarding headers against the loopback check | **7 of 7** refused |
| Three existing resource self-tests | **48 checks passed, 0 failed** |
| A new player, from the real creator through to a saved outfit and a relog | **50 of 50** |
| Three relogs per run, watching the HUD's loop, what is drawn on screen and client errors | **21 of 21, twice.** 13 of 21 on the unfixed code |
| 28 bad character payloads straight to the server, plus valid ones at the limits | **47 of 47** |
| 22 UI layouts measured for overflow, overlap and clipping, then checked in the real game | **Clean, after fixes** |
| 45 reward events across 7 heist resources, fired from nowhere near the heist | **1 real payout, fixed** |

## Bugs it found in shipped code

All three were in our shared bridge resource, were already live, passed that resource's own self-tests, and are now fixed.

1. **An unload event fired twice on every logout.** The bridge fired `QBCore:Client:OnPlayerUnload`, and then the framework's own `Logout` fired it again. 24 resources on the server listen for it, so each ran its unload logic twice. Measured 2 before the fix, 1 after.
2. **A documented export could never work.** A function passed through a FiveM export arrives as a callable table, not a function, so `type(fn) == 'function'` rejected every other resource. If your export takes a callback, check for `__cfx_functionReference` as well.
3. **Callbacks broke after a restart.** `lib.callback.register` adds a net handler and can't remove one. Registering again after the owning resource restarted left the dead handler in place, it answered first, and the caller got nothing back. Use one stable dispatcher per name.

Number 3 was hidden behind number 2, which is a good argument for testing the fix as well as the bug.

## Round two: a character screen, in the real game

The owner reported two things: new characters got the wrong apartment screen, and many UI layouts looked broken. Both were real.

- **The wrong screen.** A housing resource opens its own picker when a loaded player has no apartment. We now take the pick ourselves, claim the apartment first, and only then announce the player as loaded, so its listener finds the row and stays quiet. That needed no framework edit.
- **NaN passed validation.** `height < min or height > max` is false for NaN, because every comparison against NaN is. A modified client can send it, since it survives msgpack. Check `value ~= value` too.
- **A cancelled flow's timer fired later.** A 150-second watchdog from an abandoned handoff tore down the next scene, leaving the camera behind the player's head. Bump a generation number when a new flow starts.
- **The same watchdog punished slow players.** Anyone taking over 150 seconds to choose lost the mouse.
- **A client that could say "I finished".** One heist checked completion before paying, but the event that marks completion believed any client, so two events paid out in full. A guard is only as good as whatever sets the flag.
- **Layouts:** buttons 190 px off screen, text running over its neighbour, a badge on top of a name, labels cut off behind tiles. One cause is worth knowing: a `relative` class from a component's root beats an `absolute` you pass in, and the element lands somewhere else with no error.

## Round three: the owner sent one screenshot

Square tiles drawn on top of each other, with the question: why didn't you see this? The honest answers were more useful than the fix.

- **We only tested the default.** The layout's spacing nests hexagons, and players can switch tiles to squares or circles. "Every shape and finish" was on our list and never got run. The audit now runs all of them, at several resolutions, and saves a picture of each.
- **The overlap check compared text with text.** Controls overlapping controls were invisible to it. A hexagon's button box is still a rectangle, so the check has to look at the painted shape.
- **Shading stopped at the 16:9 stage,** leaving bright bands on ultrawide and 5:4. No metric caught that. Looking at the pictures did.
- **Each UI was tested alone.** In the same in-game shot, a status HUD sat under the Play button. A UI tested on its own can't see another resource drawing over it, so we added a check that lists what every frame really draws.

That HUD turned into the worst bug of the round. `qbx_core` empties its player data on logout just before the logged-in flag flips, and three loops read it in that gap. **An error inside a FiveM thread ends that thread for the session.** It's a race, so it's intermittent: the run that caught it survived two logouts, then the HUD froze on screen for good. Handcuff enforcement and evidence discovery had the same flaw.

## Checks that lied to us

This is the most useful part of the report. Each of these gave a clean-looking result that was wrong.

1. **One input blinded the rest.** Sending `1e308` pushed a balance to infinity, and nothing could change it after that. A wide-open handler then reported "OK" 23 times. A fuzzer needs to rerun its normal case at the end and notice when state has stopped moving.
2. **A "no change" check that could never fail.** Reading the "before" value after firing the event just records the changed value as the baseline. Make the action an argument, so the order can't be wrong.
3. **A test that ran nothing looked like a pass.** Console-only commands fired from a script refuse silently: 0 passed and 0 failed. Zero checks should never read as success.
4. **Unfinished spawn looked like desync.** Character loaded isn't the same as in the world. Clients sitting on a spawn selector see each other invisible at `z = -190`, and every sync check fails for a reason unrelated to the code under test.
5. **Inspecting the wrong client.** The DevTools port follows launch order, not which client you think it is. Have each client announce itself.
6. **`ok and x or y`.** A step returning `false` came back as the string `"false"`, because Lua falls through to `y` when `x` is false.
7. **A restart reported success with half the server down.** Predicting dependents from manifests missed real ones. Compare what was running before and after instead.
8. **A window that wasn't really 16:9.** We asked for 960x540 and got 960x561, because the outer window won't go under 600 px tall. The NUI stayed at 1920x1080 and was squashed by about 4%, and every number we'd asked for still looked right. A person watching the screen caught it. Measure the size you got, not the one you asked for.

9. **"Refused" 20 times in a row, with no client connected.** The client had been kicked, so nothing reached the server, and the database correctly showed no change. A send has to fail loudly when its sender is gone, and a set of valid inputs has to get through as well.
10. **A clean run that wasn't.** An input loop from an earlier repro was still running in that client session, drawing a radar nobody asked for. Only a fresh client gave a real result, and a check for that loop stayed in the proof.
11. **`undefined === undefined`.** Two checks passed because both sides were missing.
12. **Testing UI on a black page.** Every layout looked fine on black. Over a capture from the real game, 55 text elements fell below 3:1 contrast, the worst at 1.1:1. If your UI sits over a game world, test it over a game world.
13. **Our own fix was wrong.** Logging a player out of a spawn selector passed every proof, and a read-only code review then showed it saved the ped's preview position as their last location. Refusing that logout was the right fix. A review from someone who didn't write the code earned its cost.

14. **Our own check said a dead loop was fine.** We saw those logout errors early, compared how often each resource woke before and after a relog, saw no drop, and told the owner they did no harm. That resource has other threads, so its rate held up with its main loop dead. Ask the thing itself: listen for what the loop is telling its page, and treat any new client script error as a failure.
15. **A control that could pass on nothing.** The first version of that new check accepted an empty result in its own control line. It also needed to fail on the unfixed code before we believed it, and it did.

## Correction

An earlier version of this report said native aliases starting with `_` can't be called, and that `SetPedHairColor` throws in game. **That was wrong.** We took it from an old note and never tested it. Measured in game on client and server, it exists and works, as an alias of `SetPedHairTint`. Our natives lookup was also reporting valid aliases as not found, which made the note look true. Both are fixed.

## Wrong diagnoses

We blamed number 1 above on JSON turning integers into floats. The next check disproved it. It sounded completely reasonable, which is the point.

We also spent a long time on a gunshot test that fired but did no damage, and blamed the spawn bug. That explained the sync failures only. Then the person watching the game said the gun really was firing, so we measured where the bullets landed: **`TaskShootAtEntity` put them 38 m from a target standing 2 m away.** A bullet with a computed path from `ShootSingleBulletBetweenCoords` landed first time, and the whole hit pipeline worked. A real observation beat several rounds of reasoning.

## What is still limited

- **The rejection tests only mean something next to a control.** Forged hit reports were refused all along, but until a real hit landed, a broken pipeline would have passed too. Write the control first.
- **184 hostile inputs against the PvP events showed no crashes and no money changes.** The normal arguments don't move those values either, so this says less than the number suggests. The two-player suite is what proves the hit logic.
- **The UI audit covers one screen type well.** Character screens run through every shape, finish and several sizes. Settings and delete dialogs, and the spawn selector at other sizes, don't yet.
- **Only Qbox was run.** The fixes were read against QBCore's source, not run on it. ESX wasn't tested.
- **Flood numbers are five runs on one PC.**
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

- About 4 GB RAM per client and 0.5 GB for the server. VRAM reached 3.8 of 4 GB.
- Automated tests don't need good graphics. Going from two 1080p borderless windows with no FPS cap to two 1024x576 windows with `VSync` 2 and the lowest settings took game CPU from 53% of the PC to 29 to 35%, and GPU from 87% to 35 to 67%. RAM and VRAM barely moved. All the same tests passed.
- `gta5_settings.xml` in `%APPDATA%\CitizenFX` is one file, shared by both clients and your own game. Back it up and restore it.
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

### Per-resource CPU and memory

- Resmon has no script API, so its numbers can't be read directly.
- `profiler record N` then `profiler saveJSON` works on the server, and gives `tick (resource)` frames to add up. On the client the same commands ran and no file ever appeared, by script or by DevCon.
- For the client, we add a small script to each resource through its manifest, on the test checkout only. It wraps `Wait`, `CreateThread` and event handlers, and reads `collectgarbage('count')`.
- The client's Lua has no `os` library. `GetGameTimer` and `GetNetworkTime` don't move within a frame. `GetNetworkTimeAccurate` does, in 1 ms steps, which still adds up correctly over thousands of samples.
- Counting wake-ups per second is the easiest way to catch a tight `Wait(0)` loop.
- DevTools screenshots with `omitBackground` can be laid over a `screencapture` frame, which gives you what the player actually sees.

### Traps from long runs

- An idle test client gets kicked by an AFK timer, partway through whatever you're testing.
- A resource's client retries opening its UI a few seconds after a restart. Log in too soon and that retry can log you back out.
- FiveM sometimes hangs at "Analyzing game data" when relaunched soon after a quit. Another quit and launch cleared it.

### Network events

Five runs on one PC, so treat them as ballpark.

| Events sent | Per tick | Arrived | Result |
|---|---|---|---|
| 100 | 1 | 100 | Fine |
| 500 | 5 | 241 to 293 | About half lost, client stayed connected |
| 2000 | 20 | 123 to 142 | Client kicked: "Reliable network event overflow" |

Through msgpack, a trailing nil argument is dropped, so the argument count falls. A nil in the middle stays as a hole. NaN, inf and -inf all arrive intact.

## Credits

The RCON client and log lookup are adapted from [ziyacivan/fivem-mcp](https://github.com/ziyacivan/fivem-mcp), and the DevCon frame format came from reading it (MIT). Real keyboard input is adapted from the same project. The crash reader and resource checker are adapted from juliantsu's FiveM MCP (MIT), and the pre-flight check and NUI console capture are ideas from DoluTattoo's dolu_fivem_mcp (MIT). The Lua VM is [citizenfx/lua](https://github.com/citizenfx/lua). Built with Claude Code.
