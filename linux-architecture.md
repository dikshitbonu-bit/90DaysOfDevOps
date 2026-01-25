LINUX ARCHITECTURE: 
  1) Linux Kernel: Think of it as the brain of linux os , acts as a bridge between users and hardware,What the kernel does:

    Process management (create, schedule, kill processes)

    Memory management (RAM, swap, virtual memory)

    File systems (ext4, xfs, permissions)

    Networking (TCP/IP stack)

    Device drivers (disk, keyboard, NIC)

2) Shell: contains
            Shells: bash, zsh

            Core utilities: ls, cp, grep

            Libraries: glibc

3) Applications : ngnix,docker etc are present in this level 

WHAT IS A PROCESS? : everything running in linux is a PROCESS so think of it like a running program

HOW IS A PROCESS CREATED? : First who creates a process: only the linux kernel 
                                  who requests for a process : users via system calls

      Linux process creation = fork → exec → exit
      
      When you run a command like ls, Linux does not run the ls program directly; instead, the already-running shell (for example, bash) first asks the kernel to create a new process using fork(), which duplicates the shell and gives the child a new PID. That child process then calls exec(), which replaces the shell’s code inside the child with the ls program loaded from disk (the PID stays the same). The ls program runs, prints the output, then calls exit(), after which the kernel cleans up the process and returns control to the parent shell, which continues running and waits for the next command.

      Linux separates process creation from program execution so the shell can control, prepare, and manage programs safely, efficiently, and flexibly.

HOW IS A PROCESS CONTROLLED? :
    Linux keeps processes alive, scheduled, controlled, and killed

     Once a process is created, the kernel owns it completely.

     Each process has

     PID – process ID

     PPID – parent process ID

     State – running, sleeping, zombie, etc.

     Priority & nice value

     Memory mappings

    file descriptors

    User / group ownership

    CPU time accounting
      

    Scheduling (CPU control)

    Kernel scheduler decides:

   Which  process runs

   For how long (time slice)

    Uses priority + nice value

    Nice range:

       -20 (highest priority)
        0 (default)
        +19 (lowest priority)

   Kernel communicates with processes using signals.

    Common signals:

    SIGTERM (15) → polite stop

    SIGKILL (9) → force kill

    SIGSTOP → pause

    SIGCONT → resume

     ALL RELEVANT PROCESS MANAGEMENT COMMANDS: 
   
    ps
    ps aux
    ps -ef
    pstree
    top
    htop
    
PROCESS STATES:
     
     R – Running:
          The process is either currently using the CPU or ready to run and waiting for CPU time.
          It may switch in and out rapidly due to scheduling.
          Seen when a process is actively executing instructions.
    S – Sleeping:
         The process is waiting for something like user input, disk I/O, or a timer.
         It can be woken up by signals (like SIGTERM).
        Most normal background processes stay in this state.      
    T – Stopped / Traced
        The process is paused, usually by a signal like SIGSTOP or Ctrl+Z.
        It is not running and not eligible for CPU scheduling.
        Common during debugging or job control.    
    Z – Zombie
        The process has finished execution, but its parent has not collected its exit status.
        It holds only a PID and minimal info—no memory or CPU usage.
        Too many zombies indicate a buggy parent process.


 5 commands I would use daily:

 ps

Shows a snapshot of currently running processes at the moment the command is executed.

htop

Displays an interactive, real-time view of processes, CPU, memory usage, and allows killing processes with keys.

free -h

Shows system memory usage (RAM + swap) in human-readable format (MB/GB).

df -h

Displays disk space usage of mounted filesystems in human-readable format.

systemctl

Controls and queries systemd-managed services, including starting, stopping, enabling, and checking service status.

WHAT IS SYSTEMD? :

systemd is the modern init system in Linux that runs as PID 1 and starts after the kernel boots. It is responsible for starting, stopping, and supervising system services, handling dependencies, and enabling faster parallel booting. systemd continuously monitors services and restarts them if they fail, improving system reliability. Its importance lies in making Linux systems stable, manageable, and production-ready for servers and cloud environments.