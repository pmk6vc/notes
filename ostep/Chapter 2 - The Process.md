# Process

## Key Questions
1. How does the OS virtualize a finite number of CPUs to provide the illusion of an infinite supply of CPUs available to each running program? 

## Key Terms / Concepts
- **Process**: A running program with a *machine state* (i.e. what the program can read and update). The machine state of a process includes its virtual memory, registers (including special registers like the program counter and stack pointer), and I/O information (e.g. the files that the running process has open).  
- **Mechanism**: Low-level methods and protocols needed to perform some functionality (e.g. context switching, pausing and resuming processes).
- **Policy**: Rules for the OS to make decisions during process execution. For example, the OS scheduling policy determines which processes should run at any given time. The OS can then execute this policy by utilizing mechanisms like suspension and context switching.   
- **Time sharing**: Virtualization technique by which a physical resource is borrowed by different processes at different times to give the illusion of running processes in parallel. For example, the CPU can be time-shared across different processes via *context switching*, so it seems like each process is running in parallel. 
- **Space sharing**: Virtualization technique by which a physical resource is split up into smaller units, where each unit is assigned to a process. For example, memory is split up and assigned to each process via virtual address spaces. Similarly, disk space is space shared since each file on the disk uses space that cannot be assigned to other files.

## Process API - High Level Operations

The following is a high-level overview of operations typically available in the Process API for an OS

- **Create**: Create new process
- **Destroy**: Destroy process
- **Wait**: Wait for a process to stop running
- **Suspend**: Pause a process
- **Resume**: Resume a paused process
- **Status**: Get info about a process

## Creating Processes from Programs
Assume the starting point is a compiled program with executable bytecode persisted to disk. The high-level steps are:

- The OS provisions memory for the process's virtual address space.
- The OS loads in the executable instructions in virtual memory. This loading can either be done *eagerly* (i.e. all at once), or *lazily* (i.e. as needed during program execution). 
- The OS provisions memory for the *stack*, which contains local variables, function parameters, and return addresses. The stack will be initialized with any arguments (e.g. to the `main` function).
- The OS provisions initial memory for the *heap*, which is used to store data structures explicitly declared by the program. The OS may need to provision more than the initial memory during runtime if necessary.
- The OS performs initialization tasks to allow the program to perform I/O (e.g. to the console or file system).
- The OS actually runs the program by jumping to the `main` function entry point and running instructions, maintaining the stack pointer and program counter for the process accordingly.

The OS is itself a program, and therefore uses data structures like any other program to do its job. One of the most important data structures in the OS is the *process list*, which is a list that keeps track of running processes.

## Process States

Common process states found in operating systems (e.g. UNIX) include:

- **Initial**: Process is being created
- **Ready**: Process is not running but is ready to run on a processor
- **Running**: Instructions for this process are running on a processor
- **Blocked**: Process cannot run until some event takes place (e.g. an I/O system call has started but not completed yet)
- **Final**: Process has exited but not yet cleaned up - useful as it allows parent processes to examine return codes to ensure the completed process was successful