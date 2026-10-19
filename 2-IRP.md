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

