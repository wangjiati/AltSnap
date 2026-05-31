# PR #743 Review Reply

> To: RamonUnch
> Re: [PR #743 — Add Desktop Peek feature](https://github.com/RamonUnch/AltSnap/pull/743)

---

## A quick note before we start

I want to be upfront: I used Claude (Anthropic's AI coding assistant) to help port the PeekDesktop logic to C, organize commits, and write the PR. I did all the manual testing on real hardware myself. The bugs you spotted — dead code, class name filters, wrong LVM_HITTEST value — those are on me, I should have reviewed the AI output more carefully.

Also, Chinese is my native language and this reply was written with translation help, so please excuse any phrasing that sounds unnatural.

---

Thanks so much for the thorough review — I really appreciate you taking the time to go through the code carefully.

## Rationale for integration

I deploy AltSnap on industrial PCs where operators have minimal computer skills. There are two concrete problems that integration solves:

1. **Training cost.** Teaching a production-line worker to press Win+D and then restore windows individually is a real training burden — it seems simple to us, but for non-technical staff it generates support calls. One click on empty desktop wallpaper to hide everything, one click to bring it all back: that's intuitive for anyone.

2. **Process reliability.** Running two WH_MOUSE_LL hook-based programs side by side means two processes to keep alive on every device. If either crashes silently, the operator doesn't know which one broke — they just know "it doesn't work anymore." A single process = a single thing to monitor.

I understand this may not be a feature you want to carry in the mainline, and I'll respect your decision either way.

## Bugs and code issues — fixed

### IsDesktopIcon() — removed

You're absolutely right about the cross-process memory access. This was dead code — I removed the call during testing (the icon check made peek unusable on icon-dense desktops) but forgot to clean up the function. Removed:

- `IsDesktopIcon()`, `FindDesktopListView()`
- `PEEK_LVHITTESTINFO` struct
- `PEEK_LVM_HITTEST` constant (you're correct: LVM_HITTEST = LVM_FIRST + 18 = 0x1012)
- All `OpenProcess` / `VirtualAllocEx` / `WriteProcessMemory` / `ReadProcessMemory` / `VirtualFreeEx` calls

### PEEK_LVM_HITTEST value

Porting error from the C# source — you're correct. Since `IsDesktopIcon` is removed entirely, this constant goes with it. If it ever needs to come back, I'd use the canonical `LVM_FIRST + 18` form.

### UWP class name filters — removed

`Windows.UI.Core.CoreWindow` and `ApplicationFrameWindow` removed from `EnumPeekWindows`. The remaining filters are only the truly stable ones (Progman, WorkerW, Shell_TrayWnd — unchanged since NT4/Win95). The enumeration already safely skips unqualified windows via `IsVisible` / `IsIconic` / `IsCloaked` checks. Removing these class-name shortcuts means, at worst, a UWP window gets minimized too — which is harmless.

### WinEvent hook — kept with explanation

I've kept the `SetWinEventHook(EVENT_SYSTEM_FOREGROUND)` but added comments clarifying it is **only installed while actively peeking** AND **only when the user has enabled "Restore on app switch"** (`PeekFlags` bit3). Without the hook, peeking works fine — the user just needs to click the desktop again to restore. The hook simply makes the experience smoother: when you Alt+Tab to another app after peeking, everything restores automatically. It is uninstalled immediately when peek ends.

## Summary of changes (latest commit)

| Change | Lines |
|--------|-------|
| Remove IsDesktopIcon + helpers | -45 |
| Remove PEEK_LVHITTESTINFO + PEEK_LVM_HITTEST | -7 |
| Remove UWP class name filters | -2 |
| Add clarifying comments on WinEvent hook | +5 |
| **Net** | **-50** |

Let me know if there are other issues you'd like me to address. Thanks again for the careful review.
