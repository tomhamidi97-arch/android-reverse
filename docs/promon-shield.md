# Reversing Promon SHIELD: From Emulator Detection to an Xposed Data-Only Patch

> This is a write-up of an authorized reverse-engineering engagement against an Android app hardened with **Promon SHIELD** (RASP / app-shielding). App names, package names, endpoints, and account data are redacted. The focus is on the concrete procedure: how to walk a crash stack back to the native detection source, how to validate hypotheses with a bit-level experiment matrix, and how to turn a throwaway root patch into a stable LSPosed module that runs entirely in-process.

## 0. Fingerprinting the SDK

Promon SHIELD has a very consistent runtime signature across samples. The features to match on first:

- Obfuscated entry classes under the `yrdei.*` namespace, e.g. `yrdei.Q`, `yrdei.j`, `yrdei.F`, `yrdei.a`.
- `Application.attachBaseContext` calls into `yrdei.a.attachBaseContext -> yrdei.Q.d -> yrdei.Q.c`, which bottoms out in a native method.
- A native library (`libpostpe.so` in this build; the name varies across SHIELD versions) heavily obfuscated with OLLVM control-flow flattening and string hiding.
- On failure the app either opens a `devicenotsupported` URL or throws internal codes like `yrdei.H: 02` and `yrdei.D: 16`.
- Two exit paths: a main-thread throw through `yrdei.j.b("02")`, and a background-thread throw through `yrdei.F.run` that reads a verdict from a pipe.

Once this chain is recognized, what you are looking at is SHIELD, not an ordinary crash.

## 1. Step One: Map the Exit Path, Don't Block It

The first instinct is to hook `kill` / `exit_group` / the URL intent. That is the wrong layer. Start with the stack:

```text
AndroidRuntime: FATAL EXCEPTION: main
AndroidRuntime: yrdei.H: 02
AndroidRuntime:     at yrdei.j.a(Unknown Source:193)
AndroidRuntime:     at yrdei.j.b(Unknown Source:0)
AndroidRuntime:     at AndroidHelper_.b(AH)
AndroidRuntime:     at yrdei.Q.c(Native Method)
AndroidRuntime:     at yrdei.Q.b(Unknown Source:0)
AndroidRuntime:     at yrdei.Q.a(Unknown Source:40)
AndroidRuntime:     at yrdei.Q.d(Unknown Source:3)
AndroidRuntime:     at yrdei.a.attachBaseContext(Unknown Source:3)
```

`yrdei.j.b("02")` is the result layer. In parallel, sideband telemetry shows:

```text
SB: detGlobals ... raw=0x1022 bits=[bit1(0x2), bit5(0x20), bit12(0x1000)] ...
ActivityTaskManager: START ... dat=https://.../devicenotsupported ...
```

That `raw` integer is the aggregated detection state; each bit corresponds to one detector class. The goal is now precise:

> Find who owns `raw`, where it is written, under what condition each bit is set, then suppress that bit at the source — do not wait for it to be written and then intercept the kill.

## 2. Step Two: Dump the Detection Input Tables at Runtime

SHIELD does not hardcode its checks in assembly. Inputs live in a set of vectors that are populated at runtime. Dump them first to see what they actually contain:

```text
detLists dump:
  P1  non-empty: goldfish/ranchu path list
  P3  empty:     begin == end
  P12 non-empty: qemu/ranchu path list
  S24 non-empty: qemu/ranchu property-name list
```

Cross-reference in IDA to map each table to its consumer function and guard offset:

```text
P12_VEC=0x72c368  P12_GUARD=0x72c380  accessed by sub_225C1C
P1_VEC =0x72c388  P1_GUARD =0x72c3a0  accessed by sub_22B5C0
P2_VEC =0x72c3a8  P2_GUARD =0x72c3c0  accessed by sub_22B5C0
P3_VEC =0x72c3c8  P3_GUARD =0x72c3e0  accessed by sub_22B5C0
```

Each table is a three-pointer struct `{begin, end, third}`; the guard is a one-bit enable. These offsets are the entire attack surface for the data-only patch later.

## 3. Step Three: Pin Down the Exact Predicate for bit5

This is the crux of the whole reverse. Take `bit5(0x20)`. It is set inside `sub_22B5C0`. The decisive assembly:

```asm
0x22b678  LDP   X21, X20, [qword_72C3A8]   ; property-name list begin/end
0x22b690  BL    sub_16AD1C                 ; build returned string buffer
0x22b698  MOV   X1, X21                    ; X1 = property-name std::string*
0x22b69c  BL    sub_52372C                 ; internal property map[name] -> string
0x22b6a0  LDR   X28, [SP+var_270]          ; returned string first qword
0x22b6a8  BL    sub_167B50                 ; tear down string
0x22b6ac  CMP   X28, X22                   ; X22 = qword_7102A8 sentinel
0x22b6b0  B.EQ  loc_22B83C                 ; equal -> bit5 set path
...
0x22b950  LDR   W8, [X19]
0x22b954  ORR   W8, W8, #0x20              ; bit5
0x22b958  STR   W8, [X19]
```

The meaning: read a system property; if its value equals the internal sentinel (meaning the property matches an emulator fingerprint), set `bit5`.

Note that `sub_22B5C0` does detection **and** initialization. It cannot be wholesale NOP'd — later native-context construction depends on it, and the app hangs. The correct move is either to neutralize only the predicate at `0x22B6AC`, or better, to clear the input vector at `qword_72C3A8` so the loop has nothing to match.

## 4. Step Four: Realize bit12 Is a Derived Bit

After clearing the P12 table, `bit12` stubbornly remained. That contradicts the obvious reading. Continue into `sub_225C1C`:

```asm
0x225c4c  BL    sub_22B5C0          ; call the bit5 detector first
0x225c50  LDR   W8, [X19]
0x225c54  TBZ   W8, #5, loc_225CC8  ; skip if bit5 not set
...
0x225cb4  ORR   W8, W8, #0x1000     ; bit12 is derived from bit5
0x225cbc  STR   W8, [X19]
```

`bit12` is raised by `bit5`. Conclusion: **suppress `bit5` and `bit12` disappears for free**; there is no need to touch the P12 table at all. Such derived bits are common in these SDKs — draw the dependency graph between bits before running any experiment matrix.

## 5. Step Five: Validate with a Controlled-Variable Matrix

Reading assembly is not enough; run experiments. For each table, clear it and set its guard, then observe `raw`:

| Mode   | P1         | P2         | P3         | P12        | raw      | Interpretation                              |
|--------|------------|------------|------------|------------|----------|---------------------------------------------|
| p2     | (original) | [0,0,0]    | (original) | (original) | 0x1200   | bit5 suppressed, bit9+bit12 remain          |
| p1p2   | [0,0,0]    | [0,0,0]    | back-half hit | (original)| 0x1020 | bit9 gone, bit5 returns via P3             |
| p12p2  | (original) | [0,0,0]    | (original) | [0,0,0]    | 0x1200   | bit9 still comes from P1                    |
| all3   | [0,0,0]    | [0,0,0]    | [0,0,0]    | [0,0,0]    | no change| yet yrdei.H: 02 still thrown                 |

The `all3` row is the turning point: all four tables empty, `raw` does not move, and the process still exits. That proves the exit decision does not look only at `raw`; there is an independent verdict channel.

## 6. Step Six: Chase the Background-Thread Verdict Pipe

Filter logcat by pid again. The exit shape has changed:

```text
E/Report: yrdei.ai: !GwAAAXYUCCZCI5UA/...
E/Report:     at yrdei.j.a(Unknown Source:35)
E/Report:     at yrdei.j.b(Unknown Source:0)
E/Report:     at yrdei.F.run(Unknown Source:27)
E/AndroidRuntime: FATAL EXCEPTION: Thread-2
```

This is no longer the main-thread `yrdei.H: 02`. It is `Thread-2` in `yrdei.F.run` reading a `!`-prefixed verdict string from an fd/pipe and handing it to `yrdei.j.b`. In the Java mapping inside `j.b`, `str.startsWith("!")` constructs a `yrdei.ai` exception handed to the default UncaughtExceptionHandler.

So a temporary observation-only pipe trace was added — no return-value modification, no syscall interception — watching `pipe2/read/readv/write/writev/dup/dup3/fcntl/close` to see who writes `!`-prefixed data into that pipe. One pitfall here: a read-all version floods the log with `/proc/<pid>/maps` text reads. Narrow it to "only print remembered pipe fds or reads whose data starts with `!`", and forget fds on `close` to avoid fd-reuse noise.

Conclusion: the verdict is computed on the native side and written into the pipe. **The real source is still the native detection state**; do not patch `j.b` or the UncaughtHandler.

## 7. Step Seven: The Presence Fields in the Native Context

Extend the monitor beyond the four input tables to the native context pointer at `qword_72C550`. Several presence fields sit on that context:

```text
shared ctx gates:
  pair0 = *(base+0x72c550)
  H02: pair0+0x440 = 0
  1c : pair0+0x8d8 = 0
  00 : pair0+0x470 = 0
```

Each field maps to one detection code (`02` / `1c` / `00`). The data-only patch targets exactly these presence fields, telling the native layer these detectors found nothing.

Timing matters: the clear must happen after the native write but before the Java consume.

## 8. Step Eight: Validate the Hypothesis with a Root Helper

Before writing the real module, do a minimal root-side validation through `/proc/<pid>/mem`. The helper does two things:

1. `pwrite` zeros `base+0x72C3A8..0x72C3BF` (the P2 vector) and writes `1` to `base+0x72C3C0` (the P2 guard).
2. `pread` reads back to verify.

Crucially, start the helper and the target app **in the same on-device script**, so a host-side `su -c` background job does not end early and miss the window. One valid run:

```text
p2_data_patch target=... pid=11766 base=0x779588c000 vec=0x7795fb83a8 guard=0x7795fb83c0 rc=0 iter=39
monitor i=0  raw=0x0    tick=0x0        done=0x0  vec=[0x0,0x0,0x0]
monitor i=31 raw=0x0    tick=0x0        done=0x1  vec=[0x0,0x0,0x0]
monitor i=32 raw=0x1200 tick=0x6a4cc7da done=0x21 vec=[0x0,0x0,0x0]
```

Reading: `raw` went from `0x1022` to `0x1200`; `bit5` is gone, proving the P2/property path is suppressed. This run only validates the hypothesis; it is not the final approach.

## 9. Step Nine: Timing Design for the LSPosed Module

With the hypothesis validated, migrate to Xposed. The hard part is timing: Xposed runs code later than a root helper. Patch too late and `raw` is already written (stale).

The timing design:

```text
initZygote
  └─ hook Runtime.loadLibrary0           ← before libpostpe.so loads
       ├─ before: prepareForPostpeLoad()
       └─ after:  patchNow()             ← synchronously patch right after load
handleLoadPackage
  └─ Application.attach.before/after     ← start the native patcher thread
```

The `loadLibrary0` hook must filter its arguments so it fires only for `libpostpe.so` and does not recurse when the module loads its own `libpostpe_patch.so`:

```java
private static boolean argsContainPostpe(Object[] args) {
    for (Object arg : args) {
        String value = String.valueOf(arg);
        if (value.contains("postpe_patch")) continue;          // avoid recursion
        if ("postpe".equals(value)
            || value.endsWith("/libpostpe.so")
            || value.endsWith("libpostpe.so")) {
            return true;
        }
    }
    return false;
}
```

## 10. Step Ten: Handle Stale Detection Cache

Even with the timing above, Xposed is still a beat behind the native first detection. `raw` and `tick` may already have been written once. So the native patcher clears these two entries as a stale-cache fix:

```c
#define OFF_RAW  0x72c220UL
#define OFF_TICK 0x72c228UL

static int clear_detection_cache(uintptr_t base) {
    uint32_t raw = 0;  read_checked(base + OFF_RAW, &raw, sizeof(raw));
    uint64_t tick = 0; read_checked(base + OFF_TICK, &tick, sizeof(tick));
    if (!raw && !tick) return 0;
    write_u32(base + OFF_RAW, 0);
    write_u64(base + OFF_TICK, 0);
    return 0;
}
```

Important: **clearing the cache is not the bypass**. It only fixes stale state written before the patch landed; the steady state still depends on the source/gate data-only patch holding `raw=0x0`.

## 11. Step Eleven: The Multiple-Base Mapping Pitfall

Patching only the first base is not enough. In the same process, `/proc/self/maps` showed two `libpostpe.so` offset-0 bases:

```text
0x7b003a3000
0x7792a1b000
```

The actually-active detection state lived in the second base. With only the first patched, the log showed `hook_cache=0x0` on base 0 but `0xff` on base 1 — detection still ran.

The fix is to enumerate every base:

```c
static int find_libpostpe_bases(uintptr_t *out, int max) {
    FILE *fp = fopen("/proc/self/maps", "re");
    int count = 0; char line[512];
    while (fgets(line, sizeof(line), fp)) {
        if (!strstr(line, "libpostpe.so")) continue;
        unsigned long start, end, off; char perms[8];
        if (sscanf(line, "%lx-%lx %7s %lx", &start, &end, perms, &off) != 4 || off != 0)
            continue;
        if (count < max) out[count++] = (uintptr_t)start;
    }
    fclose(fp);
    return count;
}
```

Apply static patch + ctx presence patch + cache clear to every base. After the fix, both bases hold `raw=0x0`.

## 12. Step Twelve: The Compressed-`so` Loading Pitfall

After adding the native patcher, the first load failed outright:

```text
dlopen failed: base.apk!/lib/arm64-v8a/libpostpe_patch.so not found
```

That error is misleading — it looks like a path or ABI problem. The real cause is that the `so` inside the APK was compressed. LSPosed's module classloader `nativeLibraryDirectories` points at `base.apk!/lib/arm64-v8a`, and loading directly from inside an APK requires the `so` to be stored uncompressed.

Gradle config:

```kotlin
packaging {
    jniLibs {
        useLegacyPackaging = false
    }
}
```

After rebuild, verify with `zipinfo`:

```text
compression method: none (stored)
```

Only then does the native patcher load.

## 13. Step Thirteen: Data-Only, Never Text

Why not just flip the branch — e.g. change `B.EQ` at `0x22b6b0` to `B.NE`? Two reasons:

1. SHIELD carries text-section integrity checks; modifying text risks lighting up a new detection bit.
2. OLLVM functions are huge; editing one branch can ripple through subsequent control flow.

So every patch is data-only:

```text
static offsets:
  OFF_P4_GUARD  0x72c340   OFF_P4_VEC  0x72c328
  OFF_P5_GUARD  0x72c360   OFF_P5_TREE 0x72c348
  OFF_P12_GUARD 0x72c380   OFF_P12_VEC 0x72c368
  OFF_P1_GUARD  0x72c3a0   OFF_P1_VEC  0x72c388
  OFF_P2_GUARD  0x72c3c0   OFF_P2_VEC  0x72c3a8
  OFF_P3_GUARD  0x72c3e0   OFF_P3_VEC  0x72c3c8
  OFF_RAW       0x72c220   OFF_TICK    0x72c228   OFF_DONE 0x72c230
  OFF_HOOK_CACHE 0x724f78

dynamic ctx gates (pair0 = *(base+0x72c550)):
  H02: pair0+0x440
  1c : pair0+0x8d8
  00 : pair0+0x470
```

Writes go through `mprotect(PROT_WRITE)` opened transiently and restored afterward; text is never touched.

## 14. Step Fourteen: The KPM Side-Effect Lesson

Midway, a KernelPatch Module was used for property hiding, and it bit hard. After an AVD snapshot restore, loading an old hide KPM crashed **every** app, not just the target. Chrome's RenderThread reported:

```text
Abort message: 'couldn't find an OpenGL ES implementation, make sure one of persist.graphics.egl, ro.hardware.egl and ro.board.platform is set'
```

The cause was in the KPM's `prop_hide.c`, which had global rules:

```c
{"ro.hardware.egl", 0, NULL}
{"ro.board.platform", 0, NULL}
```

while `is_hook_process()` filtered only by `uid > 10000`. Chrome, YouTube, and every other normal app matched; their graphics stacks could no longer read EGL properties and aborted.

Lessons:

- Property hiding must be filtered **per target process** (exact thread-group `comm` match), never by uid alone.
- Never edit `build.prop` or global properties — they affect the entire emulator.
- KPMs are fine for one-shot diagnostics (like the pipe trace earlier) but **not as a deliverable**; everything final should run in-process via LSPosed.

## 15. Step Fifteen: Convergence Check

The final validation script checks only three things: no KPM, no root helper, the process is alive:

```text
KPM=0
HELPERS=
PIDS=7470
```

Key success log (both bases patched):

```text
postpe_patch: libpostpe base=0x7b003a3000
postpe_patch: static patched i=0 base=0x7b003a3000 raw=0x0 done=0x0 hook_cache=0x0
postpe_patch: libpostpe base=0x7792a1b000
postpe_patch: static patched i=1 base=0x7792a1b000 raw=0x0 done=0x0 hook_cache=0xff
postpe_patch: 1c ctx obj=... present_old=0x40 wrote +0x8d8=0
postpe_patch: 00 ctx obj=... present_old=0x40 wrote +0x470=0
postpe_patch: sync patch i=0 base=0x7792a1b000 raw=0x0 done=0x2021 hook_cache=0xff
```

After a 60-second soak, the fatal grep is empty:

```text
E/Report            (none)
E/AndroidRuntime    (none)
yrdei.H: 02         (none)
yrdei.D: 16         (none)
yrdei.ac: 1c        (none)
yrdei.d: 00         (none)
devicenotsupported  (none)
```

That is the closed loop on the entire RASP detection chain.

## 16. Takeaways

Condensed into transferable method:

1. **Map the exit path first, then find the source.** Do not touch `kill` / `exit` / URL / exception result layers.
2. **Identify derived bits.** `bit12` comes from `bit5`; clearing `bit5` is cheaper than chasing `bit12`.
3. **Run experiment matrices.** Single-variable table clears + observing `raw` beats reading OLLVM end-to-end.
4. **Patch data only.** No text edits, no skipping init functions, no `build.prop`.
5. **Mind stale cache and multiple bases.** These two are the most common reasons a "working" patch silently does nothing.
6. **Throwaway tools are for hypotheses.** Validate with a root helper, then move to LSPosed; keep KPMs for one-shot diagnostics only.
7. **Mask logs but keep the chain.** Print OTPs as `**35`, hashes as `y4***at`; print `raw` / `done` / offsets in full.

Promon SHIELD's difficulty is not any single function; it is the coupling of native detection, Java exceptions, a background verdict thread, state caching, integrity checks, and app initialization. The response has to be equally restrained: touch only confirmed detection inputs and state, scope everything to the target process, and preserve the original init flow. That restraint is the line between "it boots for a moment" and "it ships."
