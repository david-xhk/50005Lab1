# 50005Lab1

*A README note on how I tackle​d TODO#4: Implement​ main_loop() busy wait for job creation and termination*

## Main Loop (I): Posting tasks to alive child processes
My strategy for posting a task is to search for a child process that is alive and available. If such a child process exists, the task will be posted to its job buffer and the child process will be notified using `sem_post`. If the current child process is not alive and available, then I cycle to the next child process using `i = (i+1) % number_of_processes`, which will wrap around the array of pids `children_processes` in a circular manner.

### Pseudocode
```
1   if child process i is alive
2       if child process i is available
3           update child process i's job buffer with the new task
4           notify child process i with sem_post
5           break (new task has been posted)
6       else
7           cycle i to the next one
8           go to 1
9   else // child process i is not alive
10      ...
```

## Main Loop (II): When a child process is not alive
In the case that a child process is not alive, I know that it has been terminated by an 'i' task. I have decided to respawn it using the second method in the handout, which is to respawn a child process only if there are no idle processes left. I do this in five steps:

1. First, cycle through all child processes to check for any process that are both alive and available, i.e. an *idle* process.
2. When such an idle process is found, stop searching and cycle to the next child process so that eventually, the task can be posted to that idle process.
3. If an idle process cannot be found, spawn a new child process using `fork()` and dispatch it with `job_dispatch(i)`.
4. After successfully spawning the child, update the task status of that child to 0 (it is still set to 1 as the 'i' task killed it before it could update the task status to 0) so that it can be posted with a new task.
5. Loop back to the start so that the current task can be posted to the newly-spawned child process.

### Pseudocode
```
9   else // child process i is not alive
10      initialise idle flag to 0 and loop variable j to (i+1) % number_of_processes
12      if child process j is alive and available (idle)
13          set idle flag to 1
14          go to 18
15      else
16          cycle j to the next one
17          go back to 12
18      if idle flag is 1
16          cycle i to the next one
17          go back to 1
18      else
19          spawn and dispatch new child process 
20          reset task status of child process
21          go back to 1
```

## Main Loop (III): Terminating all child processes with 'z' tasks
Finally, when all tasks in the input file have been posted, the main process will send termination signals to all child processes. To achieve this, I employ a simple strategy to each child process: busy wait if it has not completed its previous task, and post the 'z' task when it is available. In order to prevent busy waiting for killed child processes due to certain race conditions between `kill()` and `waitpid()`, I exclude child processes that have been last posted with an 'i' task.

### Pseudocode
```
1   for i = 0 up to the number of processes
2       if child process i still alive
3           if child process i's task status is 1 AND if its task type is not 'i'
4               busy wait for child process i until its task status is not 1
5           update child process i's job buffer with a 'z' task
6           notify child process i with sem_post
```
