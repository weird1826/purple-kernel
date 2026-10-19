# IRP

From the previous session, I very much preached about this one keyword called "IO packet" or similar. Well in this one, we will learn about it. Both in theory, visual, and, action. Since, I am researching on `cldflt.sys`, certain keywords within the explanation will be exclusive to that. But in general, the concept of IRP remains the same.

## Vocabulary

Building vocabulary is important so we do not come across alien keywords!

* **Cldflt.sys**: Kernel's file mini filter file which processes placeholders. Sits above NTFS drivers (in terms of altitude), and intercepts I/O on cloud-backed files.
* **Sync Provider (a.k.a. sync engine)**: User-mode process that knows how to fetch bytes from the cloud. E.g., OneDrive is a Sync Provider, any third-party service can be one. It registers a **sync root**, connects to `cldflt.sys`'s port, and answers hydration requests.
* **Sync Root**: A directory that a sync provider controls. Everything under it is the provider's ownership. `cldflt.sys` tracks this directory as a provider's instance.
* **Reparse Point**: An NTFS feature that attaches custom metadata to a file or directory. Placeholders use it to store the cloud metadata.
* **Cloud Filter API (cfapi.dll)**: User-mode surface for processes to talk to `CldFlt.sys` through the functions defined in the DLL, such as to register sync root, create a placeholder, set reparse data, connect to communication port, etc.
* **Placeholder**: A file that looks like a normal file but has no real data on disk. It is just a placeholder stub having a name, size attribute, timestamps, and meta information; the bytes live in the cloud. Microsoft describes it as a reparse point owned by the minifilter.
* **Hydration**: The bytes are fetched from the cloud to the local disk. An application opens the file, `Cldflt.sys` intercepts the I/O, asks the **sync provider** for the data, writes it to the disk with a dedicated function, and let's the original request continue.
* **Dehydration**: The bytes are discarded locally and the file goes back to being a placeholder. This the reclaiming disk space mechanism.
* **Hydrated and pinned**: The content is local and guaranteed to stay local. It will not be dehydrated automatically.
* **Stream**: `Cldflt.sys`'s internal object tracking the actual content of the file, created everytime a placeholder's state changes. Not to be confused with the user-mode file. A stream is basically a kernel-side bookkeeping for what data does that placeholder have right now and what is pending.
* **Filter Communication Port**: Named port the sync provider connects to in order to exchange messages with `CldFlt.sys`.

# Stack

A file on the disk is readable through a vertical chain of device objects. Any kind of request to that file passes through a chain of drivers - the user has no idea to it. The chain is assembled at runtime, when the volume mounts. It is not compiled beforehand.

<img width="1647" height="1185" alt="stack flow concept" src="https://github.com/user-attachments/assets/5a1f7afd-950d-4008-9bf1-469263b50565" />

## See the actual chain and altitude ordering in the system

Connect WinDBG to the test VM. Sometimes things want to act up when you are debugging so it is better to reload symbols:

```
.reload /f fltmgr.sys
```
---
* **Optional [DEBUG\]**: Run this command to check if the types actually exist

```
dt fltmgr!_GLOBALS
```
---
Next, we will load `fltkd`:
```
!load fltkd
```
We are all set now.
**List all the active filters:**
```
!fltkd.filters
```
The above command outputs the active filters list:
<img width="1917" height="270" alt="active filter list" src="https://github.com/user-attachments/assets/6b585bfa-ea24-4f8c-8cf3-5452c3f4f62b" />
In my case, the entry for `cldflt` looks like this:

```
   FLT_FILTER: ffffa46c98811010 "CldFlt" "180451"
      FLT_INSTANCE: ffffa46c9880f720 "CldFlt" "180451"
```

----
**OPTIONAL READ: ADDRESS SPECIFICS**: 
In the above output, address `ffffa46c98811010` hosts the `FLT_FILTER` structure for `Cldflt`. It is NOT the load address of the `.sys` file. It is a heap object that `FltMgr` allocates to represent the registered minifilter.

To actually inspect the filter's properties in detail, use this command by noting the struct address from above:
```
!fltkd.filter ffffa46c98811010
```
Output:

---
**Find a mounted volume and its attached filters**
```
!fldkd.volumes
```
Output:
```
kd> !fltkd.volumes

Volume List: ffffa46c9b6ce140 "Frame 0" 
   FLT_VOLUME: ffffa46c9bafe4b0 "\Device\Mup"
      FLT_INSTANCE: ffffa46c9c6189a0 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9bcdab20 "UCPD - Top Instance" "385250.5"
      FLT_INSTANCE: ffffa46c9f6d1010 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9bba97a0 "WdFilter Instance" "328010"
      FLT_INSTANCE: ffffa46c9cb53640 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9bba9050 "FileInfo" "40500"
   FLT_VOLUME: ffffa46c9bbf6010 "\Device\HarddiskVolume3"
      FLT_INSTANCE: ffffa46ca0360010 "bindflt Instance" "409800"
      FLT_INSTANCE: ffffa46c9c5af660 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9bc2b400 "UCPD - Top Instance" "385250.5"
      FLT_INSTANCE: ffffa46c9f708230 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9bd65010 "WdFilter Instance" "328010"
      FLT_INSTANCE: ffffa46c9880f720 "CldFlt" "180451"
      FLT_INSTANCE: ffffa46c98816010 "luafv" "135000"
      FLT_INSTANCE: ffffa46c9caf66e0 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9bc22990 "Wof Instance" "40700"
      FLT_INSTANCE: ffffa46c9bc258a0 "FileInfo" "40500"
   FLT_VOLUME: ffffa46c9bcbe010 "\Device\NamedPipe"
      FLT_INSTANCE: ffffa46c9c619920 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9f6248a0 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9cb7e6e0 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9be03010 "npsvctrig" "46000"
   FLT_VOLUME: ffffa46c9bcbf010 "\Device\Mailslot"
      FLT_INSTANCE: ffffa46c9c61c920 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9f7028e0 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9cb1e850 "bfs" "100010"
   FLT_VOLUME: ffffa46c9c642010 "\Device\HarddiskVolume1"
      FLT_INSTANCE: ffffa46c9c3b1010 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9c6c5b20 "UCPD - Top Instance" "385250.5"
      FLT_INSTANCE: ffffa46c9f2ee8a0 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9c67b010 "WdFilter Instance" "328010"
      FLT_INSTANCE: ffffa46c9cb1b850 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9c6c4010 "FileInfo" "40500"
   FLT_VOLUME: ffffa46c9c65b420 "\Device\HarddiskVolume4"
      FLT_INSTANCE: ffffa46c9c741960 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9c6bfb30 "UCPD - Top Instance" "385250.5"
      FLT_INSTANCE: ffffa46c9f6268a0 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9c679660 "WdFilter Instance" "328010"
      FLT_INSTANCE: ffffa46c9bede010 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9c67a010 "Wof Instance" "40700"
      FLT_INSTANCE: ffffa46c9c70e640 "FileInfo" "40500"
```
<img width="766" height="551" alt="image" src="https://github.com/user-attachments/assets/0368df64-eb02-40ae-8e16-1d6d59db264c" />

`CldFlt` is attached to HarddiskVolume3 in my case. You can see its properties using this command:

```
!object \Device\HarddiskVolumeN
```
_note: replace N with your actual number_
```
kd> !object \Device\HarddiskVolume3
Object: ffffa46c9bbb8470  Type: (ffffa46c9888a0c0) Device
    ObjectHeader: ffffa46c9bbb8440 (new version)
    HandleCount: 0  PointerCount: 28
    Directory Object: ffffd2e7ce2ad1f0  Name: HarddiskVolume3
```
We will pick the (3) with `CldFlt` attached:
```
!fltkd.volume ffffa46c9bbf6010
```
Output:
```
kd> !fltkd.volume ffffa46c9bbf6010

FLT_VOLUME: ffffa46c9bbf6010 "\Device\HarddiskVolume3"
   FLT_OBJECT: ffffa46c9bbf6010  [04000000] Volume
      RundownRef               : 0x0000000000003400 (6656)
      PointerCount             : 0x00000001 
      PrimaryLink              : [ffffa46c9bcbe020-ffffa46c9bafe4c0] 
   Frame                    : ffffa46c9b6ce010 "Frame 0" 
   Flags                    : [00000564] SetupNotifyCalled EnableNameCaching FilterAttached EnableDatascan SupportedFeaturesKnown
   FileSystemType           : [00000002] FLT_FSTYPE_NTFS
   VolumeLink               : [ffffa46c9bcbe020-ffffa46c9bafe4c0] 
   DeviceObject             : ffffa46c9bbecd60 
   DiskDeviceObject         : ffffa46c9bbb8470 
   FrameZeroVolume          : ffffa46c9bbf6010 
   VolumeInNextFrame        : 0000000000000000 
   Guid                     : "\??\Volume{f436a02c-8413-489e-820e-a66d69d02682}" 
   CDODeviceName            : "\Ntfs" 
   CDODriverName            : "\FileSystem\Ntfs" 
   TargetedOpenCount        : 6645 
   Callbacks                : (ffffa46c9bbf6158)
   ContextLock              : (ffffa46c9bbf6540)
   VolumeContexts           : (ffffa46c9bbf6550)  Count=0
   StreamListCtrls          : (ffffa46c9bbf65d8)  rCount=22743 
   FileListCtrls            : (ffffa46c9bbf6658)  rCount=6329 
   NameCacheCtrl            : (ffffa46c9bbf66d8)
   InstanceList             : (ffffa46c9bbf60d8)
      FLT_INSTANCE: ffffa46ca0360010 "bindflt Instance" "409800"
      FLT_INSTANCE: ffffa46c9c5af660 "MsSecFlt Instance" "385600"
      FLT_INSTANCE: ffffa46c9bc2b400 "UCPD - Top Instance" "385250.5"
      FLT_INSTANCE: ffffa46c9f708230 "wesp Instance" "329500"
      FLT_INSTANCE: ffffa46c9bd65010 "WdFilter Instance" "328010"
      FLT_INSTANCE: ffffa46c9880f720 "CldFlt" "180451"
      FLT_INSTANCE: ffffa46c98816010 "luafv" "135000"
      FLT_INSTANCE: ffffa46c9caf66e0 "bfs" "100010"
      FLT_INSTANCE: ffffa46c9bc22990 "Wof Instance" "40700"
      FLT_INSTANCE: ffffa46c9bc258a0 "FileInfo" "40500"
```
From the output, note down the address of `DeviceObject`, which is `ffffa46c9bbecd60` in my case. We will cross check that there are no raw devices on top of it. There is just filter manager and within it, there are filters:
```
!devstack ffffa46c9bbecd60
```
Output:
```
kd> !devstack ffffa46c9bbecd60
  !DevObj           !DrvObj            !DevExt           ObjectName
> ffffa46c9bbecd60  \FileSystem\FltMgr ffffa46c9bbeceb0  
  ffffa46c9bbe2030  \FileSystem\Ntfs   ffffa46c9bbe21b0  
```
