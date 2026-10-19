# Introduction

Drivers are a very interesting surface for security research because they handle almost everything happening in the operating system such as managing, intercepting, processing I/O which includes everything from network and hardware, liaising between the OS and the machine, and so on and so forth while running in kernel mode.

Because there is a lot going on while managing such a large stateful architecture, it is no doubt that suble bugs are almost inexistent in this realm. Low-hanging fruits are for sure gone or have gotten extremely rare, but if you know what you are doing, if you have the right tools and knowledge, and that ability to think and write notes on, then it gets just a little easier to find bugs.

This post is going to be about Filter Drivers - not exactly on how to find bugs, but it rather going to be my logbook for my own learning journey as I attempt to hunt for bugs, and learn new topics and concepts regarding filter driver research. I am excited and look forward to this and I hope that you (yes the reader) gets something out of my little learning path. Thank you very much for following and reading this thread!

# Filter Drivers

Both filter and normal drivers have almost identical architecture, but their functioning differs differently. Both run with Ring 0 Privileges, receive I/O packets (which are called IRPs - we will learn later), etc. The difference is not mechanism, it is the ownership and position in the entire architecture.

A normal driver creates and owns its own device object and sits at the bottom of responsibility for that device. For example, if you remove the driver for USB support, the USB functionality is dead. Hence, an I/O packet meant for USB is received by USB driver because it is the target.

Filter drivers own nothing. It attaches a filter device object to someone else's stack, like a volume or a file system, and intercepts I/O packets in transit. Since it can intercept these I/O requests, it can inspect, modify, complete early, or pend them. Note that filter drivers are usually optional. You can unload it and the underlying device keeps working. It is not visible to the running applications, hence, no PnP identity.

Continuing on that, a few consequences worth nothing are first, ordering matters! There is not one filter driver running, there are multiple filter drivers running in the system. In the previous paragraph, that a filter driver attaches itself to a device's stack, hence, its behaviour depends on where it sits relative to the other filters. This is known an altitude. Also typical edge cases include certain circumstances, where a filter must assume that someone above or below will do something unexpected and should have the right functionalities to handle such unexpected cases.

A specific application of filter driver can be a file system filter. For file I/O, the path looks roughly like this:

```
app -> IO Manager -> [FS filter A] -> [FS filter B] -> [FS Filter ...] -> NTFS -> volume -> disk
```

A file system filter attaches to a volume, not a device in a PnP sense. It intercepts I/O packets that IO manager builds for file-based operations. Hence, a mouse filter sees mouse traffic, a file system filter sees every file operation on the attached volume, from every process. Practically this means that any process doing a file operation passes through the filter's call-backs. There can be times when volumes mount and unmount at runtime; in such times, a filter has to be notified when a volume appears, attach an instance, setup per-volume state, and tear that state down when volume goes away. A file system filter can also hold an IO packet while it ways for something which could be a disk, network, or user mode tasks and complete it later.

Implementing a filter driver can be very challenging as we have to handle every IO request type (even those we do not care about), so that they can be forwarded to the next driver in stack. It is also important to figure out the correct point to insert our filter driver into the device stack. To make things easier, there is now a special driver called Filter Manager Driver `FltMgr` which takes care of handling IO requests and the altitude handling. This can allow developer to write a "mini-filter driver" instead of a normal one.

**Minifilters** are a further specialization of filter drivers. These do not get attached with a device manually and do not require hand-writing pass-through for every operation, or registering call-backs with `FltMgr` . Instead, `FltMgr` attaches the filter to volumes as they mouth, dispatches the operations we registered for, manages per-file/per-volume contexts, and gives us the communication port. 

These images from <a href="https://projectzero.google/2021/01/hunting-for-bugs-in-windows-mini-filter.html" target="_blank">James Forshaw's PZ article</a> are excellent to visually understand the differences between a normal filter driver and a Minifilter driver: 

<img width="653" height="502" alt="visual of a normal filter driver" src="https://github.com/user-attachments/assets/5266cba7-c231-43cc-9de4-25cd66ae744a" />
<br/>
Img:Visual of a normal filter driver
<br/>
<br/>
<br/>
<img width="820" height="507" alt="visual of a mini filter driver" src="https://github.com/user-attachments/assets/167b9ffd-9822-4e6d-a0b0-c47dd86a0340" />
<br/>
Img:Visual of a mini filter driver using filter manager
<br/>
<br/>
Notice how the mini-filter driver do not have to add its own device to the stack. Instead, simply registering with filter manager automatically solves complexities dealing with IO requests and processing call requests. If the mini-filter does not support a certain IO request, then the filter manager has its own mechanisms to pass the IO packet to the next driver in stack. 
<br/><br/>
James Forshaw also notes that for altitude, filter manager considers a higher altitude as being more prioritized. Hence, a mini-filter with altitude 10,000 will be called first than the one with altitude 5000. Natural question in priority would be what if "altitudes" clash with each other? Well, they won't and there is a reason: Microsoft publishes a table for a list of drivers with officially Microsoft-registered altitudes along with the vendor names, which includes Microsoft's own drivers itself. See here: https://learn.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes#392000---394999-fsfilter-security-monitor. Driver needs to be certified so that altitudes do not mangle up.

