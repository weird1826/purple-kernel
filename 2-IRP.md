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

# IRP Structure

An IRP is a single heap allocation with two parts:
1. A fixed header - the request as a whole: status, requestor mode, buffer, pointer, flags, cancel routine, and stack bookkeeping.
2. Variable length array of `IO_STACK_LOCATION` slots - one slot per drier in the device stack the request will traverse.

The array length is decided at allocation time from the number of device objects in the chain. That count is in the header field `StackCount`. The IO Manager allocates `sizeof(IRP) + StackCount + sizeof(IO_STACK_LOCATION)`, implies the size of the IRP is not fixed. It is obvious that every driver needs its own private parameter space. If CldFlt wants to modify the read length before passing it down, it must not stomp on what NTFS expects to see. Giving each driver its own slot solves that issue. CldFlt fills slot N+1, passes the IRP down and NTFS reads N+1 fresh.

IRP structure:

<img width="2913" height="3418" alt="image" src="https://github.com/user-attachments/assets/28ecc6a9-6a2d-4e42-910c-9f4f6e59ff28" />


In WinDBG, run this command:
```
dt nt!_IRP
```
Output:
```
kd> dt nt!_IRP
   +0x000 Type             : Int2B
   +0x002 Size             : Uint2B
   +0x004 AllocationProcessorNumber : Uint2B
   +0x006 Reserved1        : Uint2B
   +0x008 MdlAddress       : Ptr64 _MDL
   +0x010 Flags            : Uint4B
   +0x014 Reserved2        : Uint4B
   +0x018 AssociatedIrp    : <unnamed-tag>
   +0x020 ThreadListEntry  : _LIST_ENTRY
   +0x030 IoStatus         : _IO_STATUS_BLOCK
   +0x040 RequestorMode    : Char
   +0x041 PendingReturned  : UChar
   +0x042 StackCount       : Char
   +0x043 CurrentLocation  : Char
   +0x044 Cancel           : UChar
   +0x045 CancelIrql       : UChar
   +0x046 ApcEnvironment   : Char
   +0x047 AllocationFlags  : UChar
   +0x048 UserIosb         : Ptr64 _IO_STATUS_BLOCK
   +0x048 IoRingContext    : Ptr64 Void
   +0x050 UserEvent        : Ptr64 _KEVENT
   +0x058 Overlay          : <unnamed-tag>
   +0x068 CancelRoutine    : Ptr64     void 
   +0x070 UserBuffer       : Ptr64 Void
   +0x078 Tail             : <unnamed-tag>
```
Note the locations for StackCount, CurrentLocation, and Tail. I am not able to expand them manually by `_IRP.Tail`, etc. because they are possibly unions and are showing as unnamed. But we can use this command to expand all:
```
dt nt!_IRP -b
```
Output:
```
kd> dt nt!_IRP -b
   +0x000 Type             : Int2B
   +0x002 Size             : Uint2B
   +0x004 AllocationProcessorNumber : Uint2B
   +0x006 Reserved1        : Uint2B
   +0x008 MdlAddress       : Ptr64 
   +0x010 Flags            : Uint4B
   +0x014 Reserved2        : Uint4B
   +0x018 AssociatedIrp    : <unnamed-tag>
      +0x000 MasterIrp        : Ptr64 
      +0x000 IrpCount         : Int4B
      +0x000 SystemBuffer     : Ptr64 
   +0x020 ThreadListEntry  : _LIST_ENTRY
      +0x000 Flink            : Ptr64 
      +0x008 Blink            : Ptr64 
   +0x030 IoStatus         : _IO_STATUS_BLOCK
      +0x000 Status           : Int4B
      +0x000 Pointer          : Ptr64 
      +0x008 Information      : Uint8B
   +0x040 RequestorMode    : Char
   +0x041 PendingReturned  : UChar
   +0x042 StackCount       : Char
   +0x043 CurrentLocation  : Char
   +0x044 Cancel           : UChar
   +0x045 CancelIrql       : UChar
   +0x046 ApcEnvironment   : Char
   +0x047 AllocationFlags  : UChar
   +0x048 UserIosb         : Ptr64 
   +0x048 IoRingContext    : Ptr64 
   +0x050 UserEvent        : Ptr64 
   +0x058 Overlay          : <unnamed-tag>
      +0x000 AsynchronousParameters : <unnamed-tag>
         +0x000 UserApcRoutine   : Ptr64 
         +0x000 IssuingProcess   : Ptr64 
         +0x008 UserApcContext   : Ptr64 
         +0x008 IoRing           : Ptr64 
      +0x000 AllocationSize   : _LARGE_INTEGER
         +0x000 LowPart          : Uint4B
         +0x004 HighPart         : Int4B
         +0x000 u                : <unnamed-tag>
            +0x000 LowPart          : Uint4B
            +0x004 HighPart         : Int4B
         +0x000 QuadPart         : Int8B
   +0x068 CancelRoutine    : Ptr64 
   +0x070 UserBuffer       : Ptr64 
   +0x078 Tail             : <unnamed-tag>
      +0x000 Overlay          : <unnamed-tag>
         +0x000 DeviceQueueEntry : _KDEVICE_QUEUE_ENTRY
            +0x000 DeviceListEntry  : _LIST_ENTRY
               +0x000 Flink            : Ptr64 
               +0x008 Blink            : Ptr64 
            +0x010 SortKey          : Uint4B
            +0x014 Inserted         : UChar
         +0x000 DriverContext    : Ptr64 
         +0x020 Thread           : Ptr64 
         +0x028 AuxiliaryBuffer  : Ptr64 
         +0x030 ListEntry        : _LIST_ENTRY
            +0x000 Flink            : Ptr64 
            +0x008 Blink            : Ptr64 
         +0x040 CurrentStackLocation : Ptr64 
         +0x040 PacketType       : Uint4B
         +0x048 OriginalFileObject : Ptr64 
         +0x050 IrpExtension     : Ptr64 
      +0x000 Apc              : _KAPC
         +0x000 Type             : UChar
         +0x001 AllFlags         : UChar
         +0x001 CallbackDataContext : Pos 0, 1 Bit
         +0x001 Unused           : Pos 1, 7 Bits
         +0x002 Size             : UChar
         +0x003 SpareByte1       : UChar
         +0x004 SpareLong0       : Uint4B
         +0x008 Thread           : Ptr64 
         +0x010 ApcListEntry     : _LIST_ENTRY
            +0x000 Flink            : Ptr64 
            +0x008 Blink            : Ptr64 
         +0x020 KernelRoutine    : Ptr64 
         +0x028 RundownRoutine   : Ptr64 
         +0x030 NormalRoutine    : Ptr64 
         +0x020 Reserved         : Ptr64 
         +0x038 NormalContext    : Ptr64 
         +0x040 SystemArgument1  : Ptr64 
         +0x048 SystemArgument2  : Ptr64 
         +0x050 ApcStateIndex    : Char
         +0x051 ApcMode          : Char
         +0x052 Inserted         : UChar
      +0x000 CompletionKey    : Ptr64 
kd> g
```
Note that in this build, `CurrentStackLocation = IRP + 0x78 + 0x000 + 0x040 = IRP + 0xb8`.
Next, we will attempt to dump a live IRP. A syscall like `NtReadFile` proceeds to call `IofCallDriver(PDEVICEOBJECT, PIRP)` from where, IRP is construced. So we put a breakpoint at `IofCallDriver` and dump its rdx (since it is the second argument).
```
kd> !irp rdx
Irp is active with 9 stacks 10 is current (= 0xffffa46ca17f2a08)
 No Mdl: No System Buffer: Thread ffffa46c9dbdc080:  Irp is completed.  
     cmd  flg cl Device   File     Completion-Context
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000    

			Args: 00000000 00000000 00000000 00000000
 [IRP_MJ_CREATE(0), N/A(0)]
            0  0 ffffa46c9bbb28f0 ffffa46ca67baee0 00000000-00000000    
	       \Driver\iorate
			Args: ffffc9de0b9803f8 01000060 00030000 00000000
```

There is a chance that you can get an output like this. The stack numbers and current numbers might differ. In my capture, it is 9 stacks and 10 is current. Implies, this is the sentinel state. The address in brackets `(= 0xffffa46ca17f2a08)` is `Tail.Overlay.CurrentStackLocation`, pointing one slot past the end of the array. We are at `IoFCallDriver` entry, before `IoSetNextIrpStackLocation` is called internally.

Notice slots 0 to 7 are all zeroes. Slot 8 is `StackCount - 1` which is the topmost slot. That is the first driver that will receive this CREATE request and it was filled by IO Manager just before it called `IofCallDriver`. The rest of the lines above are waiting to be populated as the request travels down.

But if necessary, I will get back to it. If the numbers are low, and if you are seeing things like `[N/A(e), N/A(0)]` all along, then it is better to repeat this to get an IRP with a higher number. We will try to dump the IRP at rdx at this point:

```
kd> dt nt!_IRP ffffa46ca17f26b0
   +0x000 Type             : 0n6
   +0x002 Size             : 0x478
   +0x004 AllocationProcessorNumber : 0
   +0x006 Reserved1        : 0xf805
   +0x008 MdlAddress       : (null) 
   +0x010 Flags            : 0x884
   +0x014 Reserved2        : 0xc88edf39
   +0x018 AssociatedIrp    : <unnamed-tag>
   +0x020 ThreadListEntry  : _LIST_ENTRY [ 0xffffa46c`9dbdc5c0 - 0xffffa46c`9dbdc5c0 ]
   +0x030 IoStatus         : _IO_STATUS_BLOCK
   +0x040 RequestorMode    : 1 ''
   +0x041 PendingReturned  : 0 ''
   +0x042 StackCount       : 9 ''
   +0x043 CurrentLocation  : 10 ''
   +0x044 Cancel           : 0 ''
   +0x045 CancelIrql       : 0 ''
   +0x046 ApcEnvironment   : 0 ''
   +0x047 AllocationFlags  : 0x4 ''
   +0x048 UserIosb         : 0xffffc9de`0b9803c8 _IO_STATUS_BLOCK
   +0x048 IoRingContext    : 0xffffc9de`0b9803c8 Void
   +0x050 UserEvent        : (null) 
   +0x058 Overlay          : <unnamed-tag>
   +0x068 CancelRoutine    : (null) 
   +0x070 UserBuffer       : (null) 
   +0x078 Tail             : <unnamed-tag
```

See that the size is 0x478. Size of `IO_STACK_LOCATION` struct (each slot) is 0x48 in this build, which one can confirm using this command:

```
kd> dt nt!_IO_STACK_LOCATION
   +0x000 MajorFunction    : UChar
   +0x001 MinorFunction    : UChar
   +0x002 Flags            : UChar
   +0x003 Control          : UChar
   +0x008 Parameters       : <unnamed-tag>
   +0x028 DeviceObject     : Ptr64 _DEVICE_OBJECT
   +0x030 FileObject       : Ptr64 _FILE_OBJECT
   +0x038 CompletionRoutine : Ptr64     long 
   +0x040 Context          : Ptr64 Void
```

Continuing on that, there are 9 slots. So, 9 * 0x48 = 0x288 for all slots. The `CurrentStackLocation` is `0xffffa46ca17f2a08` and IRP Base is `0xffffa46ca17f26b0` (the rdx value). The difference between these two is 0x358 and that is where the sentinel lives. Slot 0 starts at `base + 0x358 - 9*0x48`, where 9 comes from the number of slots, and 0x48 is the size of each slot. This boils down to `base + 0xd0`. So, `size of IRP + StackCount * 0x48 = 0xd0 + 0x288 = 0x358`, while Size is `0x478`. The extra `0x120` is IO manager's reserved space and not the part of the struct or the slot array.

Hence, takeaways from this is that IofCallDriver fires on every IO in the system. 

# `IO_STACK_LOCATION` Slot

We will understand the slot for each driver that it receives. Now you see, each `IO_STACK_LOCATION` in an IRP is one driver's private parameter block. It is written by the **driver above** (or by the IO Manager for the top slot), read by the driver it is addressed to, and then either passed down (with the next slot filled) or completed.

The slot has six things that matter:
1. `MajorFunction`: IRP major code which tells driver what _kind_ of request this is.
2. `MinorFunction`: a sub-code for operations that have one. `CREATE` uses `IRP_MN_*`; most others have `IRP_MN_NORMAL=0`.
3. `Flags`/`Control`: `SL_INVOKE_ON_SUCCESS/ERROR/CANCEL` (when the completion routine should run), and `SL_PENDING_RETURNED` (whether the op has been pended).
4. `Parameters`: It is a union data type and its layout depends on `MajorFunction`. The arguments are placed here.
5. `DeviceObject`: The device object this slot is targeting. Usually the device this current driver is attached to.
6. `FileObject`: The `FILE_OBJECT` struct for file-related requests. The same field is valid for CREATE, READ, WRITE, CLEANUP, CLOSE, and the information/control ops. If we are looking for "which file", this is where we can find it, regardless of major code.
7. `CompletionRoutine` / `Context`: This is the driver's completion call back and its argument, if it registered one with `IoSetCompletionRoutine`.

All union members occupy the same bytes. Reading `.Read.Length` off a `CREATE` IRP reads bytes that happen to belong to `.Create.ShareAccess` or padding; the value is meaningless. But this is a bug class which appears in such drivers: a handler that reads the wrong union member for the operation it is processing, or one that trusts a class/control value from user mode and then reinterprets the same buffer as a different structure.

For `cldflt` specifically, the union members that matters are the following:

| MajorFunction | Union member | Fields that matter |
|---|---|---|
| `IRP_MJ_CREATE` | `Parameters.Create` | `SecurityContext`, `Options` (disposition flags like `FILE_OPEN`, `FILE_CREATE`), `FileAttributes`, `ShareAccess`, `EaLength`. The file name is on `FILE_OBJECT->FileName`, not here. `FILE_OBJECT->RelatedFileObject` matters for relative opens. |
| `IRP_MJ_READ` | `Parameters.Read` | `Length`, `Key`, `ByteOffset` (`LARGE_INTEGER`) |
| `IRP_MJ_WRITE` | `Parameters.Write` | `Length`, `Key`, `ByteOffset`, `WriteCompletionContext` |
| `IRP_MJ_CLEANUP` | *(none)* | Last handle is closing; no parameters. The file object is the payload. |
| `IRP_MJ_CLOSE` | *(none)* | Object is going away. Same. |
| `IRP_MJ_QUERY_INFORMATION` | `Parameters.QueryFileInformation` | `Length`, `FileInformationClass`, `FileInformation` |
| `IRP_MJ_SET_INFORMATION` | `Parameters.SetFileInformation` | `Length`, `FileInformationClass`, `FileInformation` |
| `IRP_MJ_FILE_SYSTEM_CONTROL` | `Parameters.FileSystemControl` | `OutputBufferLength`, `InputBufferLength`, `FsControlCode`, `Type3InputBuffer` |
| `IRP_MJ_DEVICE_CONTROL` | `Parameters.DeviceIoControl` | `OutputBufferLength`, `InputBufferLength`, `IoControlCode`, `Type3InputBuffer` |

You might be asking that we discussing all about these file operations but are not seeing the actual pointer to the data anywhere. For instance, look at the row for `IRP_MJ_READ`:

```
Parameters.Read.Length    : ULONG
Parameters.Read.Key       : ULONG
Parameters.Read.ByteOffset: LARGE_INTEGER
```
One might think that the Length is the size of the data, but where is the pointer to the data itself? Shouldn't there be a buffer field here? Well, no. The slot specifies how much data to move. IRP has the information of data's location.

<img width="2207" height="738" alt="image" src="https://github.com/user-attachments/assets/3580fe72-e15c-49fe-a632-c7dd10b20609" />

The slot specifies the quantity and IRP says location. Hence, the slot is per-driver and the buffer is per-request.

The reason such a setup is there is because imagine a read request going through Cldflt to NTFS to disk. If each driver had its own buffer pointer in its own slot, the developer would have three pointers to the same bugger, and the rules for who owns the pointer, who can change it, and what happens if one driver redirects the buffer would be a nightmare. Hence, one buffer on the IRP is a good choice,. 

There are two these two methods: `IRP_MJ_QUERY_INFORMATION` and `IRP_MJ_SET_INFORMATION`. Both of these have a field called `FileInformationClass`. For these two operations, the buffer's layout is not fixed. The same buffer bytes mean different things depending on what `FileInformationClass` says they are. It is a type tag and it conveys which struct to cast the buffer to. On Windows there are dozens of these classes. Here are three examples:

<img width="1599" height="2005" alt="image" src="https://github.com/user-attachments/assets/308d515f-5714-47b0-b9b4-ef618159a757" />

Because both the class number and buffer bytes are supplied by the caller, a driver has to do three things correctly:
1. Validate that the class is one it knows how to handle. If driver cannot handle a class number 47, the driver must reject it and not to fall-through and read the buffer with the wrong struct.
2. Validate that length is big enough for the struct that class implies. Class 10, which is `FileRenameInformation` has a variable length name at the end. If the length is 0x10 but the caller claims class 10, a driver that reads `FileNameLength` from the buffer and then `FileName` beyond the buffer boundary is reading out of bounds. This is exactly the pattern in many file system vulnerabilities.
3. Not re-read the class later and assume it is the same: If the driver validates class 4, then passes the IRP down, and something below changes the class field to 10, then a later driver reading the buffer as class 10 gets type confusion.

Here is the complete structure of `IO_STACK_LOCATION`:

<img width="2173" height="2758" alt="image" src="https://github.com/user-attachments/assets/ebbcbb2e-4a42-45e6-a9f0-1234cf990762" />

The union is one 0x20 byte region. Every named member aliases the same memory. That is the whole game.
