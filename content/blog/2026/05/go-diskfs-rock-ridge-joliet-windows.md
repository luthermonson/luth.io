---
title: "go-diskfs: Getting Rock Ridge Working for Joliet on Windows"
description: "Fixing Rock Ridge, adding Joliet, and then finding out what Windows actually wanted, all so a Go-built cloud-init ISO works for cloudbase-init."
date: 2026-05-17T20:40:00-07:00
tags: [ go, iso9660, rock-ridge, joliet, go-diskfs, cloud-init, windows, open-source ]
categories: [ go, windows ]
images:
 - src: img/2026/05/go-diskfs-rock-ridge-joliet-windows-layout.svg
   alt: "Sector map of a cidata ISO built by go-diskfs with a Rock Ridge primary tree and a Joliet supplementary tree sharing the same file extents"
   stretch: "none"
draft: false
---

[go-proxmox](https://github.com/luthermonson/go-proxmox) has had a `CloudInit()` helper since 2023. It builds a small
nocloud ISO in Go, uploads it to Proxmox and mounts it on the VM as a CD-ROM. It's always worked fine for Linux guests. It
has never worked for Windows guests running [cloudbase-init](https://cloudbase.it/cloudbase-init/), and the reason is a
filesystem extension from 1995 called Joliet.

Getting it fixed took three PRs to [go-diskfs](https://github.com/diskfs/go-diskfs) over about seven weeks, plus a bug report
from a user who actually ran it on Windows, which turned out to be the most important part.

### Why Windows can't read the ISO

Plain ISO 9660 is very strict about names. At interchange level 1 a file name is up to 8 characters of uppercase
letters, digits and underscore, a dot, a 3 character extension and a `;1` version suffix. Here are the three nocloud files
as go-diskfs writes them into the primary directory tree:

```
META_DAT.;1
NETWORK_.;1
USER_DAT.;1
```

Nobody actually reads those names. There are two extensions that layer real filenames on top, and which one an OS uses
decides what it sees:

- **Rock Ridge** is for Unix. It hides POSIX metadata inside each directory record's System Use area using the SUSP
  framework: `PX` for mode/uid/gid/links, `NM` for the real name, `SL` for symlink targets, `TF` for timestamps, `CL`/`PL`/`RE`
  for relocating directories nested deeper than ISO 9660's 8-level limit, and an `ER` entry announcing the extension.
  Linux reads it, so Linux sees `meta-data`.
- **Joliet** is Microsoft's. It doesn't touch the primary tree at all. It adds a Supplementary Volume Descriptor, or SVD
  (sector 17 here, right behind the PVD), flagged with an escape sequence (`%/@`, `%/C` or `%/E` for UCS-2 levels 1 to 3), pointing at a
  *second* directory tree whose names are UCS-2 big-endian, up to 64 characters.

Windows' CD-ROM filesystem doesn't do Rock Ridge. Without a Joliet tree it sees `META_DAT.;1`, and cloudbase-init, which is
looking for `meta-data`, finds nothing. Here's the layout of the ISO that works, with both trees, read out of a real one:

![Sector map of a go-diskfs cidata ISO with Rock Ridge and Joliet](/img/2026/05/go-diskfs-rock-ridge-joliet-windows-layout.svg)

### There is no Go library for this

I went looking before writing anything. As of March:

- [kdomanski/iso9660](https://github.com/kdomanski/iso9660): its README says it plainly, "Joliet extension is **NOT**
  supported", with Rock Ridge reading still experimental.
- [hooklift/iso9660](https://github.com/hooklift/iso9660): a reader only, last touched in 2019.
- [bgrewell/iso-kit](https://github.com/bgrewell/iso-kit): new (January 2025), a couple of stars, says it does Joliet but calls itself early
  development.
- [go-diskfs](https://github.com/diskfs/go-diskfs): the serious one, and what go-proxmox already used. It could write
  Rock Ridge, and "Joliet extensions to iso9660" was sitting in the plans section of the README, with
  [issue #73](https://github.com/diskfs/go-diskfs/issues/73) asking for it open since 2020.

So, go-diskfs. The two extensions are independent on disk (the maintainer pointed that out in review, and he's right), but Rock Ridge was already in the tree and half-working. Before bolting a second directory tree onto the
finalize code, I wanted the first one to be correct.

### Step 1: Rock Ridge up to spec (#353)

[PR #353](https://github.com/diskfs/go-diskfs/pull/353), merged April 10th. Most of the fixes were small and each one is a
good example of the spec being specific about something the code had been loose about:

- **Modes were hardcoded.** `Mode()` returned `0o755` for everything regardless of the `PX` entry. Now it reads the `PX`
  mode and only falls back to fixed values for non-Rock Ridge images:

  ```go
  func (de *directoryEntry) Mode() os.FileMode {
      for _, ext := range de.extensions {
          if px, ok := ext.(rockRidgePosixAttributes); ok {
              return px.mode
          }
      }
      // Fallback for non-Rock Ridge
      if de.isSubdirectory {
          return os.ModeDir | 0o755
      }
      // ... symlink fallback
      return 0o755
  }
  ```

- **No way to get at the metadata.** `Sys()` now returns a `*RockRidgeInfo` with UID, GID, link count, mode and symlink
  target, which is the standard Go way to expose filesystem-specific detail through `fs.FileInfo`:

  ```go
  type RockRidgeInfo struct {
      UID     uint32
      GID     uint32
      Nlink   uint32
      Mode    os.FileMode
      Symlink string
  }
  ```

- **`RE` was 8 bytes.** The Relocated Directory entry is just a marker: 2 bytes of signature, a length byte and a version
  byte, so its length is 4. The code allocated 8 and zero-filled the rest. Harmless to most readers, still wrong.

- **Relocated directories were looked up too late.** When a tree is deeper than 8 levels, the deep directory is moved
  and a `CL` (child link) entry points at its new home. The reader checked path depth before resolving the `CL`, so relocated
  directories didn't resolve at every level. The lookup moved ahead of the depth check.

- **Symlinks were treated as files during finalize.** A symlink's target lives inline in the `SL` entry; it has no data
  extent. But finalize put symlinks in the list of files to copy, so `os.Open()` would follow the link and either fail on a
  dangling one or copy the target's bytes. They're now excluded from data allocation while still getting directory entries.

- **Relative symlinks came out absolute.** The `SL` parser glued components together with a leading separator, so
  a relative `target.txt` came back as `/target.txt`. The fix only adds a separator once something is already in the path, and review asked for that
  repeated check to be pulled into a helper:

  ```go
  appendComponent := func(component string) {
      if name != "" && name != "/" {
          name += "/"
      }
      name += component
  }
  ```

The more interesting part was testing. I wanted to check go-diskfs against a reference implementation instead of against
itself, and the reference for Rock Ridge is [xorriso](https://www.gnu.org/software/xorriso/). The PR as first written added
Docker-based tests in both directions (build with go-diskfs and check with `isoinfo`, build with xorriso and read with
go-diskfs) plus
a byte-level comparison of the `PX` and `NM` entries each one writes. Getting xorriso into the test image hit a fun Alpine
problem: `cdrkit` (which provides `isoinfo`) and `xorriso` both provide `mkisofs` and refuse to install together. My
workaround involved shuffling binaries around and editing `/etc/apk/world`, which got a well-deserved "why the whole
rigamarole" in review. The maintainer dropped cdrkit from the image entirely and I rewrote the test to read back with
go-diskfs instead of `isoinfo`. Much simpler.

Reading the metadata back looks like this:

```go
entries, err := fs.ReadDir(".")
if err != nil {
    log.Fatal(err)
}
for _, e := range entries {
    fi, _ := e.Info()
    rr, _ := fi.Sys().(*iso9660.RockRidgeInfo)
    fmt.Printf("%-16s %v size=%d rr=%+v\n", fi.Name(), fi.Mode(), fi.Size(), rr)
}
```

```
meta-data        -rw-rw-rw- size=49 rr=&{UID:0 GID:0 Nlink:0 Mode:-rw-rw-rw- Symlink:}
network-config   -rw-rw-rw- size=46 rr=&{UID:0 GID:0 Nlink:0 Mode:-rw-rw-rw- Symlink:}
user-data        -rw-rw-rw- size=37 rr=&{UID:0 GID:0 Nlink:0 Mode:-rw-rw-rw- Symlink:}
```

(That ISO was written on Windows, hence `0666`, and go-diskfs's Windows stat shim returns zero for link count, UID and GID,
which is on my list.)

### Step 2: Joliet (#372)

[PR #372](https://github.com/diskfs/go-diskfs/pull/372) went up the day #353 merged and landed April 16th, closing #73 after
more than five years. On the write side it's a single flag:

```go
err = fs.Finalize(iso9660.FinalizeOptions{
    RockRidge:        true,
    Joliet:           true,
    VolumeIdentifier: "cidata",
})
```

Under that flag, finalize builds a parallel directory tree with UCS-2 names, generates Joliet L and M path tables, and writes
an SVD with the UCS-2 level 3 escape sequence. Both trees point at the same file extents, so the data is only on disk once. On
the read side go-diskfs detects a Joliet SVD by its escape sequence and transparently reads the Joliet tree when Rock Ridge
isn't active.

Two review questions are worth repeating because they're the kind of thing the spec makes easy to get wrong. First, Rock
Ridge and Joliet aren't mutually exclusive, and a comment in my code claiming "Joliet entries never have SUSP" read like they
were. The flag is per directory entry: entries in the Joliet tree carry no SUSP, entries in the primary tree carry the full
Rock Ridge set. Second, with both El Torito and Joliet enabled, does the root directory location get bumped twice? Yes,
because the boot record and the SVD are each their own volume descriptor sector, and that deserved a comment.

go-proxmox switched `makeCloudInitISO()` over to `Joliet: true` and shipped it in
[v0.5.0](https://github.com/luthermonson/go-proxmox/releases/tag/v0.5.0) on May 7th, pinned to go-diskfs v1.9.2.

### Step 3: what Windows actually wanted (#391)

A day later [go-proxmox#258](https://github.com/luthermonson/go-proxmox/issues/258) came in from
[rgl](https://github.com/rgl), titled "0.5.0 creates an invalid iso file". On Ubuntu, `blkid` showed this as the label:

```
/dev/sr0: BLOCK_SIZE="2048" UUID="2026-05-08-21-11-46-82" LABEL="M-fM-^MM-)M-fM-^QM-!M-gM-^QM-!" TYPE="iso9660"
```

That's `cat -v` escaping for 9 bytes of UTF-8, and if you decode them you get three CJK characters: 捩摡瑡. Which is what you
get when you take the six ASCII bytes of `cidata`, `63 69 64 61 74 61`, and read them as UCS-2 big-endian: `0x6369`,
`0x6461`, `0x7461`.

```
$ printf 'cidata' | iconv -f UCS-2BE -t UTF-8 | cat -v
M-fM-^MM-)M-fM-^QM-!M-gM-^QM-!
```

The Joliet SVD's volume identifier is a UCS-2 field like everything else in that descriptor, and go-diskfs was copying the
ASCII string straight in. Linux preferred the Joliet label, got garbage, and cloud-init's nocloud datasource never found a
volume called `cidata`. So the Joliet work had broken *Linux* guests too, which is the opposite of the plan.

Fixing the label got Linux back, and on Windows the label now showed correctly and `dir D:\` listed the files. But reading
one failed with `Cannot find path 'D:\meta-data' because it does not exist`, while an xorriso-built ISO
worked. The second bug was in the Joliet path table. Every directory gets a path table record, and the root's identifier
is a single byte. I'd written `0x01`. ECMA-119 says the
root's path table identifier is a single `0x00` byte, in the Joliet tables as much as the primary ones. My best read is that Windows resolves paths through the path table, so it could enumerate the root through its directory
record but couldn't resolve a file path through the bad table entry.

[PR #391](https://github.com/diskfs/go-diskfs/pull/391) fixes both. The identifier encoding is gated on the escape sequence,
since review pointed out that not every SVD is a Joliet SVD:

```go
func (v *supplementaryVolumeDescriptor) toBytes() []byte {
    b := volumeDescriptorFirstBytes(volumeDescriptorSupplementary)

    b[7] = v.volumeFlags
    if isJolietSVD(v) {
        copy(b[8:40], ucs2StringToBytes(v.systemIdentifier))
        copy(b[40:72], ucs2StringToBytes(v.volumeIdentifier))
    } else {
        copy(b[8:40], v.systemIdentifier)
        copy(b[40:72], v.volumeIdentifier)
    }
    // ...
}
```

and the path table root becomes `"\x00"`. rgl confirmed it worked against a Windows guest running cloudbase-init, the maintainer
tagged go-diskfs v1.9.3 on May 14th, and go-proxmox v0.5.1 pinned it the same day.

### Why the tests didn't catch it

I had round-trip tests and xorriso tests, and neither caught a wrong label. The round-trip tests couldn't: the reader and writer
had the same bug in opposite directions. `parseSupplementaryVolumeDescriptor()` read the identifier as raw bytes and
`toBytes()` wrote raw bytes, so a go-diskfs ISO read back by go-diskfs said `cidata` every time. A round trip through your own
code proves your reader and writer agree with each other. It says nothing about whether either one agrees with the spec.

The thing that caught it was someone booting a real VM. I'd add tests that assert specific bytes against the spec (the
SVD identifier is UCS-2, the root path table entry is `00`) instead of only asserting that what goes in comes back out.

### How I tested

To check this for the post I built the same three-file ISO (`RockRidge: true, Joliet: true, VolumeIdentifier: "cidata"`) with
go-diskfs v1.9.2 and v1.9.3 and compared them on Windows 11 (build 26200) with `Mount-DiskImage`:

```
v1.9.2  SVD volume id bytes: 63 69 64 61 74 61 00 00 ...
        Joliet L path table root: 01 00 1a 00 00 00 01 00 01 00
        Get-Volume label: 捩摡瑡   dir D:\ lists 3 files   Get-Content D:\meta-data -> Cannot find path

v1.9.3  SVD volume id bytes: 00 63 00 69 00 64 00 61 00 74 00 61 ...
        Joliet L path table root: 01 00 1a 00 00 00 01 00 00 00
        Get-Volume label: cidata   Get-Content D:\meta-data -> instance-id: vm-100 ...
```

Same symptoms rgl saw in the VM. One extra thing turned up: the raw files are 57,354 bytes while the primary volume
descriptor claims 29 sectors (59,392 bytes), and `Mount-DiskImage` refused both as "corrupted and unreadable" until I padded
them to 29 full sectors. Proxmox and QEMU don't care, but it's another thing for the list.

### Links

- [diskfs/go-diskfs#353](https://github.com/diskfs/go-diskfs/pull/353) - Rock Ridge support
- [diskfs/go-diskfs#372](https://github.com/diskfs/go-diskfs/pull/372) - Joliet support
- [diskfs/go-diskfs#391](https://github.com/diskfs/go-diskfs/pull/391) - Joliet SVD UCS-2 volume identifier and path table root
- [luthermonson/go-proxmox#258](https://github.com/luthermonson/go-proxmox/issues/258) - "0.5.0 creates an invalid iso file"
- [rgl/proxmox-clone-vm-example](https://github.com/rgl/proxmox-clone-vm-example) - rgl's Windows + cloudbase-init example
- [ECMA-119](https://ecma-international.org/publications-and-standards/standards/ecma-119/) - the ISO 9660 standard
- [Rock Ridge Interchange Protocol (IEEE P1282)](http://web.archive.org/web/20071017082049/http://www.nextcomputers.org/NeXTfiles/Projects/CD-ROM/Rock_Ridge_Interchange_Protocol.pdf)
- [Joliet Specification](https://pismotec.com/cfs/jolspec.html)
