# Chapter 5: Processes, Threads, and Jobs

## Process Internals

### Data Structures

- Each Windows process is represented by an **executive** process (`EPROCESS`) structure.
- Each process has one or more threads, each represented by an executive thread (`ETHREAD`) structure.
- The `EPROCESS` and most of its related data structures exist in **system** address space. One exception is the **PEB**, which exists in the **process** address space (because it contains information accessed by user-mode code).
- Some of the process data structures used in memory management, such as the **working set list**, are valid only within the context of the **current process**, because they are stored in **process-specific** system space.
- For each process that is executing a Win32 program, the Win32 subsystem process (`Csrss`) maintains a parallel structure called the `CSR_PROCESS`.
- Finally, the kernel-mode part of the Win32 subsystem (`Win32k.sys`) maintains a per-process data structure, `W32PROCESS`.
- Every `EPROCESS` structure is encapsulated as a process object by the **executive** object manager.
<p align="center"><img src="./assets/process-thread-data-structures.png" width="300px" height="auto"></p>

- The first member of the executive process structure is called **Pcb**.
- It is a structure of type `KPROCESS`, for kernel process. Although routines in the **executive** store information in the `EPROCESS`, the **dispatcher**, **scheduler**, and **interrupt/time** accounting code - being part of the OS kernel - use the `KPROCESS` instead.
- This allows a layer of abstraction to exist between the executive’s high-level functionality and its underlying low-level implementation of certain functions, and it helps prevent unwanted **dependencies** between the layers.
<p align="center"><img src="./assets/eprocess.png" width="400px" height="auto"></p>

- The PEB lives in the user-mode address space of the process it describes. It contains information needed by the **image loader**, the **heap manager**, and other Windows components that need to access it from user mode.
- The `EPROCESS` and `KPROCESS` structures are accessible only from kernel mode. The important fields of the PEB are illustrated below:
<p align="center"><img src="./assets/peb-fields.png" width="400px" height="auto"></p>

- bEcause each session has its own instance of the Windows subsystem, the `CSR_PROCESS` structures are maintained by the *Csrss* process within each individual session.
<p align="center"><img src="./assets/csr_process.png" width="400px" height="auto"></p>

- You can dump the `CSR_PROCESS` structure with the `!dp` command in the `user`-mode debugger.
  - The `!dp` command takes as input the **PID** of the process whose `CSR_PROCESS` structure should be dumped
  - Alternatively, the structure pointer can be given directly as an argument Because `!dp` already performs a dt command internally, there is no need to use `dt` on your own: `0:000> !dp v 0x1c0aa8-8`
- The `W32PROCESS` structure contains all the information that the **Windows graphics** and **window management** code in
the kernel (Win32k) needs to maintain state information about GUI processes.
<p align="center"><img src="./assets/w32_process.png" width="300px" height="auto"></p>

- There is no command provided by the debugger extensions to dump the `W32PROCESS` structure, but it is present in the symbols of the Win32k driver.
  - As such, by using the `dt` command with the appropriate symbol name `win32k!_W32PROCESS`, it is possible to dump the
fields as long as the pointer is known.
  - Because the `!process` does not actually output this pointer (even though it is stored in the `EPROCESS` object), the field must be inspected manually with dt `nt!_EPROCESS Win32Process` followed by an `EPROCESS` pointer.

## Protected Processes

- In the Windows security model, any process running with a token containing the **debug** privilege (such as an **administrator**’s account) can request **any** access right that it desires to any other process running on the machine.
  - For example, it can RW **arbitrary** process memory, inject code, suspend and resume threads, and query information on other processes 🤸‍♀️.
- This logical behavior (which helps ensure that administrators will always have full control of the running code on the system) **clashes** with the system behavior for DRM requirements imposed by the media industry 🤷.
- To support reliable and protected playback of such content, Windows uses **protected processes**.
- The OS will allow a process to be protected only if the image file has been **digitally signed** with a special Windows Media Certificate ‼️
- Example of protected processes:
  - The Audio Device Graph process (`Audiodg.exe`) because protected music content can be decoded through it.
  - WER client process (`Werfault.exe`) can also run protected because it needs to have access to protected processes in case one of them crashes.
  - The System process itself is protected because some of the decryption information is generated by the` Ksecdd.sys` driver and stored in its UM memory. Additionally, it protects the integrity of all kernel **handles**.
- At the kernel level, support for protected processes is twofold:
  1. The bulk of process creation occurs in KM to avoid injection attacks.
  2. Protected processes have a special bit set in their `EPROCESS` structure that modifies the behavior of security-related routines in the process manager to deny certain access rights that would normally be granted to administrators. In fact, the only access rights that are granted for protected processes are `PROCESS_QUERY`/`SET_LIMITED_INFORMATION`, `PROCESS_TERMINATE`, and `PROCESS_SUSPEND_RESUME`. Certain access rights are also disabled for threads running inside protected processes.

## Flow of CreateProcess

- A Windows subsystem process is created when a call is make to one of the process-creation functions, such as `CreateProcess`, `CreateProcessAsUser`, `CreateProcessWithTokenW`, or `CreateProcessWithLogonW`.
- Here is an overview of the stages Windows follows to create a process:
  1. **Validate parameters**; convert Windows subsystem flags and options to their native counterparts; parse, validate, and convert the attribute list to its native counterpart.
  2. Open the image file (.exe) to be executed inside the process.
  3. Create the Windows **executive** process object.
  4. Create the **initial thread** (stack, context, and Windows **executive** thread object)
  5. Perform post-creation, **Windows-subsystem-specific** process initialization.
  6. Start execution of the initial thread (unless the `CREATE_SUSPENDED` flag was specified).
  7. In the context of the new process and thread, complete the initialization of the address space (such as load **required DLLs**) and begin execution of the program.
<p align="center"><img src="./assets/process-creation-stages.png" width="400px" height="auto"></p>

### Stage 1: Converting and Validating Parameters and Flags

- You can specify more than one **priority class** for a single `CreateProcess` call but Windows chooses the lowest-priority class set.
- If no priority class is specified for the new process, the priority class defaults to **Normal** unless the priority class of the process that created it is **Idle** or **Below Normal**, in which case the priority class of the new process will have the same priority as the creating class.
- If a **Real-time** priority class is specified for the new process and the process’ caller doesn’t have the *Increase Scheduling Priority* privilege, the **High** priority class is used instead.
- If no desktop is specified in `CreateProcess`, the process is associated with the caller’s **current desktop**.
- If the creation flags specify that the process will be **debugged**, `Kernel32` initiates a connection to the native debugging code in `Ntdll.dll` by calling `DbgUiConnectToDbg` and gets a handle to the debug object from the current TEB.
- The user-specified attribute list is **converted** from Windows subsystem format to **native** format and internal attributes are added to it.

<details><summary>Process Attributes:</summary>


| Native Attribute |  Equivalent Windows Attribute | Type | Description |
|------------------|-------------------------------|------|-------------|
| PS_CP_PARENT_PROCESS | PROC_THREAD_ATTRIBUTE_PARENT_PROCESS. Also used when elevating | Input | Handle to the parent process |
| PS_CP_DEBUG_OBJECT   | N/A – used when using DEBUG_PROCESS as a flag | Input | Debug object if process is being started debugged |
| PS_CP_PRIMARY_TOKEN  | N/A – used when using `CreateProcessAsUser/WithToken` | Input | Process token if `CreateProcessAsUser` was used |
| PS_CP_CLIENT_ID      | N/A – returned by Win32 API as a parameter | Output | Returns the TID and PID of the initial thread and the process |
| PS_CP_TEB_ADDRESS    | N/A – internally used and not exposed | Output |  Returns the address of the TEB for the initial thread |
| PS_CP_FILENAME       | N/A – used as a parameter in `CreateProcess` API | Input | Name of the process that should be created |
| PS_CP_IMAGE_INFO     | N/A – internally used and not exposed | Output | Returns `SECTION_IMAGE_INFORMATION`, which contains information on the version, flags, and subsystem of the executable, as well as the stack size and entry point |
| PS_CP_MEM_RESERVE    | N/A – internally used by SMSS and CSRSS | Input | Array of virtual memory reservations that should be made during initial process address space creation, allowing guaranteed availability because no other allocations have taken place yet |
| PS_CP_PRIORITY_CLASS | N/A – passed in as a parameter to the CreateProcess API | Input | Priority class that the process should be given |
| PS_CP_ERROR_MODE     | N/A – passed in through `CREATE_DEFAULT_ERROR_MODE` flag | Input | Hard error-processing mode for the process |
| PS_CP_STD_HANDLE_INFO| | Input | Specifies if standard handles should be duplicated, or if new handles should be created |
| PS_CP_HANDLE_LIST    | `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` | Input | List of handles belonging to the parent process that should be inherited by the new process|
| PS_CP_GROUP_AFFINITY | `PROC_THREAD_ATTRIBUTE_GROUP_AFFINITY` | Input | Processor group(s) the thread should be allowed to run on |
| PS_CP_PREFERRED_NODE | `PROC_THREAD_ATTRIBUTES_PRFERRED_NODE` | Input | Preferred (ideal) node that should be associated with the process. It affects the node on which the initial process heap and thread stack will be created |
| PS_CP_IDEAL_PROCESSOR| `PROC_THREAD_ATTTRIBUTE_IDEAL_PROCESSOR` | Input | Preferred (ideal) processor that the thread should be scheduled on |
| PS_CP_UMS_THREAD     | `PROC_THREAD_ATTRIBUTE_UMS_THREAD` | Input | Contains the UMS attributes, completion list, and context |
| PS_CP_EXECUTE_OPTIONS| `PROC_THREAD_MITIGATION_POLICY` | Input | Contains information on which mitigations (SEHOP, ATL Emulation, NX) should be enabled/disabled for the process |

</details>

### Stage 2: Opening the Image to Be Executed

- The first stage in `NtCreateUserProcess` is to find the appropriate Windows image that will run the executable file specified by the caller and to create a section object to later map it into the address space of the new process.
<p align="center"><img src="./assets/windows-image-to-activate.png" width="300px" height="auto"></p>

- Now that `NtCreateUserProcess` has found a valid Windows executable image, it looks in the registry under `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options` to see whether a subkey with the file name and extension of the executable image exists there.
  - If it does, `PspAllocateProcess` looks for a value named *Debugger* for that key. If this value is present, the image to be run becomes the string in that value and `CreateProcess` restarts at Stage 1.

<details><summary>Decision Tree for Stage 1 of CreateProcess:</summary>

| If the Image ... | Create State Code | This Image Will Run |  ... and This Will Happen |
| ---------------- | ----------------- | ------------------- | ------------------------- |
| Is a POSIX executable file | PsCreateSuccess | `Posix.exe` | CreateProcess restarts Stage 1 |
| Is an MS-DOS application with an exe, com, or pif extension | PsCreateFailOnSectionCreate | `Ntvdm.exe` | CreateProcess restarts Stage 1 |
| Is a Win16 application | PsCreateFailOnSectionCreate | `Ntvdm.exe` | CreateProcess restarts Stage 1|
| Is a Win64 application on a 32-bit system (or a PPC, MIPS, or Alpha Binary) | PsCreateFailMachineMismatch  | N/A | CreateProcess will fail |
| Has a Debugger key with another image name | PsCreateFailExeName | Name specified in the Debugger key | CreateProcess restarts Stage 1 |
| Is an invalid or damaged Windows EXE | PsCreateFailExeFormat | N/A | CreateProcess will fail Cannot be opened PsCreateFailOnFileOpen N/A CreateProcess will fail |
| Is a command procedure (application with a bat or cmd extension) | PsCreateFailOnSectionCreate | `Cmd.exe` | CreateProcess restarts Stage 1 |

</details>
