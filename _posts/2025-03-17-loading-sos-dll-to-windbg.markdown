---
layout: post
title: "Loading sos.dll to Windbg for .NET Core: The Correct Way"
date: 2025-10-02 11:09:50 +0200
categories: [debugging, windbg, dotnet, memory-dump]
tags: [sos, dotnet-core, windbg, debugging]
---

# Loading sos.dll to Windbg

**Windbg** is still the golden standard to troubleshoot memory dumps, and the **sos extension** is still the essential tool to help with managed memory dump investigation.

With **.NET Framework** (4.8 or earlier), loading it could be as easy as:

```

.loadby sos clr

```

But things got a little trickier with **.NET Core**, as you need to install `dotnet-sos` for the extension, and that could cause some confusion to get it to work.

---

## The Common Mistake: Installing as a Global Tool

The thing is that you can't (or rather, shouldn't stop at) installing `dotnet-sos` like this:

```

dotnet tool install -g dotnet-sos

```

Which will install it in a *weird* path, similar to:
`C:\Users\quma\.dotnet\tools\.store\dotnet-sos\9.0.607501\dotnet-sos\9.0.607501\tools\net6.0\win-x64`

And if you run the command to load it using a similar path:

```

.load C:\\Users\\vimvq.dotnet\\tools.store\\dotnet-sos\\8.0.510501\\dotnet-sos\\8.0.510501\\tools\\net6.0\\any\\win-x64\\sos.dll

```

You can't run any command you're used to, like `!dumpheap`:

```

0:000\> \!dumpheap
Error: Fail to create host ldelegate 80070002
Error: ICLRRuntimeHost::ExecuteInDefaultAppDomain failed 80070002
Unrecognized command 'dumpheap'

```

---

## Why the Error Occurs

The reason is that now `sos.dll` is simply a **loader**. Commands like `!dumpheap` are implemented in a different DLL that is **not in the same folder** with `sos.dll`.

If you were to open the folder installed via `dotnet tool install -g dotnet-sos`, you would see you don't have the assemblies that contain the commands you need.

---

## The Correct Approach: Using `dotnet-sos install`

What you want is to install it with the specific installation command:

```

dotnet-sos install

```

This command will install it in a more convenient and fully configured path, like:
`C:\Users\quma\.dotnet\sos`

And the folder will contain all the necessary assemblies.

Then, you can run the correct `.load` command in WinDBG:

```

.load C:\\Users\\quma.dotnet\\sos\\sos.dll

```

Everything should run as expected!

***
**Note:** If you ran the `.load` command with the wrong path before, you need to **restart the debugging process** (Press **Stop Debugging** and **Ctrl+D** to reload the memory dump).
```