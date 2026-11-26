# Analyze and clean up the Windows component store

The Windows component store (`C:\Windows\WinSxS`) holds Windows components and
files needed to service or repair Windows. Its size in File Explorer can be
misleading because some files are hard-linked elsewhere. Use DISM to check how
much space the store can actually reclaim and, if appropriate, remove superseded
component versions.

## Before you start

- Save your work and use an **Administrator** PowerShell or Windows Terminal
  window. To open one, right-click **Start** and select **Terminal (Admin)** or
  **Windows PowerShell (Admin)**, then approve the User Account Control prompt.
- Run these commands while Windows is running; `/Online` means the current
  Windows installation.
- Let servicing finish before starting, and keep the computer powered on while
  cleanup is running. It can take several minutes.

## 1. Analyze the component store

Run:

```powershell
DISM /Online /Cleanup-Image /AnalyzeComponentStore
```

This checks the component store and prints a report. Look for:

- **Component Store Cleanup Recommended** — whether DISM recommends cleanup.
- **Backups and Disabled Features** — space associated with backups and
  disabled features.
- **Number of Reclaimable Packages** — superseded package versions that may be
  removed.

If cleanup is not recommended or there is little reclaimable space, you may
stop here.

## 2. Start component cleanup

If cleanup is recommended, run:

```powershell
DISM /Online /Cleanup-Image /StartComponentCleanup
```

This removes superseded versions of components from the component store. It
does not delete personal files or act as a general temporary-file cleanup. Wait
for DISM to finish and report that the operation completed successfully; do not
close the terminal or power off the computer while it is working.

## 3. Check the result

You can run the analysis command again to see the updated report:

```powershell
DISM /Online /Cleanup-Image /AnalyzeComponentStore
```

The amount reclaimed depends on the system, and cleanup may not produce a large
change.

## Important

Do not add `/ResetBase` unless you specifically intend to: it removes all
superseded component versions and prevents uninstalling existing Windows
updates. The commands above do not use `/ResetBase`.
