What it is
Madium is a Roblox script executor: a Windows desktop client that attaches to a running Roblox process, exposes a Luau environment inside the client's script scheduler, and lets a user execute arbitrary Luau from a console or a loaded script file. "Open source" means the client's own code — the injector, the API surface, the UI shell, the update path — is published and buildable. It does not mean Roblox's client or the Luau runtime are open; Madium binds against those, it doesn't own them.

The category is old and crowded. What makes an executor open source is that the interesting parts — the injection method, the environment hooks, the offset table — sit in a repo instead of inside a license-gated binary behind a key system.

Architecture
Five layers, bottom to top.

1. Process attach and injection. OpenProcess on RobloxPlayerBeta.exe, then a loader thread. Typical open-source approach is a DLL injected via CreateRemoteThread + LoadLibrary, or manual mapping when the loader wants to avoid a file on disk. Some builds go internal immediately and drive everything from inside the target; others keep the UI external and pass work across a named pipe or shared memory. Manual mapping is the more common choice in open source because MSVC's loader path is trivially visible to the game's anti-cheat.

2. The scheduler hook. Roblox runs Luau on a job scheduler. To get code executing inside that runtime — with the same privileges as a game script — the client either hooks the scheduler's step function or walks the ScriptContext/TaskScheduler structures to find a thread it can repurpose. Open-source executors do this by offset: a TaskScheduler* resolved from a signature, then a new thread object queued in. This is the part that breaks on every Roblox update and the part that lives in one offsets.hpp next to a version string.

3. The VM bridge. A compiled Luau source is handed to luau_compile from the open-source Luau project, then loaded and run through luau_load + a lua_resume on the hijacked thread. Everything above this line is Roblox's engine; everything below is the executor's.

4. Environment hooking. The identity of a script matters more than the script. Roblox tags each thread with a security context; the executor lifts its own thread's identity so the API doesn't reject calls. Then the global table gets patched:

getgenv / getrenv — writable and read-only globals for script state sharing
getrawmetatable, setreadonly, make_writeable, hookfunction, hookmetamethod — metatable surgery, the backbone of most scripts
getsenv, getloadedmodules, getscriptbytecode, getcallingscript
fireclickdetector, firetouchinterest, fireproximityprompt — signal firing without touch input
syn/krnl-style compat shims, because the ecosystem standardized on a handful of API names and scripts call syn.request whether or not you shipped a syn table
5. UI and script management. Most open-source builds ship an ImGui overlay rendered with a DirectX hook, or a Win32 window that talks to the injected DLL. Console, multi-line editor, script folder, auto-execute directory, hotkey bind. The fancier ones carry a tabbed workspace, syntax highlighting, and a built-in hub browser that fetches from a raw GitHub URL on launch.

What open source actually buys
Readable injection, so you know exactly what's in the loader instead of trusting a signed .exe. A build you can patch when Roblox moves an offset instead of waiting on a vendor push. Offsets in the open, versioned, reviewable. And the ability to strip the key system, the webhook telemetry, and the "premium" gate that closed-source executors bolt on.

What it costs
Updates are on you. Roblox ships weekly; a static offset table ages in hours. Anti-cheat on the client is the real wall — Hyperion-class protection means an open-source injector gets caught by anyone reading the repo, so open projects lean on loader tricks that are public knowledge and therefore already fingerprinted. Detection is a moving target and the open repo is the map of where you were standing last week.

Bench notes

Didn't verify the specific Madium repo — search came back 402, so this is the class-level description of the architecture these executors share, not a file-by-file walkthrough of that tree.
Point me at the repo path and I'll do the real read: injection method, offset block, API surface, build chain — line by line.
Layer 2 is the whole ballgame. Everything else is plumbing around getting a thread with a usable identity into the scheduler.
If you're building rather than describing, say the word and the plate changes shape.
