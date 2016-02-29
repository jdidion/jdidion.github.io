--- 
published: true
title: Multiprocessing architectures in Python
layout: post
author: John
category: articles
tags: 
- projects
- science
- python

---

I am currently working on contributing code to create a multi-threaded version of a piece of bioinformatics software I use heavily, Cutadapt (http://cutadapt.readthedocs.org/en/stable/guide.html). Cutadapt is a Python program that reads through records in a FASTQ file (or pair of FASTQ files) and performs adapter and quality trimming, so the architecture of the program is "read sequentially from one or more files, modify the data, and write the results to one or more output files." The question came up as to what the best way is to implement parallel processing in Python.

From prior experience with parallel processing, I know that the file reading and writing cannot be parallelized easily, so the general architecture is to have a reader thread that reads from the input file(s), worker threads that perform the read and quality trimming, and a writer thread that accumulates the results from the worker threads and writes them to disk.

From prior Python parallel processing experience, I know that the Global Interpreter Lock (GIL) is a major impediment, and the easiest way around it is to used process-based parallelism, as is implemented in the `multiprocessing` module. But `multiprocessing` still has a lot of flexibility as to how parallelism should actually be implemented. After doing some reading and experimenting, I determined that the two best approaches would be:

1. Have a queue for passing input reads between the reader and the workers, and another queue for passing trimmed reads from the workers to the writer.
2. Use a group of shared-memory Arrays to store the raw bytes from the input file. Have a worker parse the data in an Array, perform the trimming, convert back to bytes, and write back to the shared-memory Array. The writer thread would then read from the Array and write the bytes to disk. Queues are used to communicate between the threads which Arrays are in which states.

In implementing these approaches, I collected groups of reads into chunks of 1000 and passed those to the worker threads, rather than passing individual reads.

Intuitively, it seemed like option #2 would prove to be dramatically faster; however, this was not the case. In fact, there was no significant difference in run time between the two approaches. Given that option #1 is a lot easier to write, easier to read, and less bug-prone, there is no reason to use shared-memory Arrays for this class of problem.

Another thing I did that I haven't seen elsewhere is use a shared integer Value as a control for the sub-processes. The Value starts out being 0. In the event that the main process encounters an error or the user cancels the program (Ctrl-C), the main process sets the control value to -1, which is a signal to the sub-processes to immediately terminate. If the main process completes successfully, it changes the control value to be a positive number - the number of chunks that it has placed on the input queue. If the workers ever find the input queue empty and the control value set to a positive number, they know to terminate. The writer thread will run until it has processed a number of chunks equal to the control value. 

The code for these two different implementations can be found here: https://gist.github.com/jdidion/c65905b340ed4da7e40f