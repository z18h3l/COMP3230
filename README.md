java cCOMP3230 Principles of Operating Systems
Programming Assignment Two
Due date: November 17 , 2024, at 23:59
Total 12 points – RC2
Programming Exercise – Accelerate LLM Inference using Multi-Threading
Objectives
1. An assessment task related to ILO 4 [Practicability] – “demonstrate knowledge in applying system 
software and tools available in the modern operating system for software development”.
2. A learning activity related to ILO 2.
3. The goals of this programming exercise are:
• to have hands-on practice in designing and developing multi-threading programs;
• to learn how to use POSIX Pthreads (and Semaphore) libraries to create, manage, and 
coordinate multiple threads in a shared memory environment;
• to design and implementsynchronization schemes for multithreaded processes using 
semaphores, or mutex locks and condition variables.
Tasks
Optimizing the throughput of GPTs is an important topic. Similar to other neural networks, GPT and
its variations utilize matrix-vector-multiplication, or called fully-connected/linear layer in Deep 
Learning, to apply parameters learned. Meanwhile, GPT leverages multi-head attention, a 
mechanism to adopt important information from history tokens. Thus, to accelerate GPT and get a 
faster response, it’s critical to have faster matrix-vector-multiplication and faster multi-head 
attention computation. Given the non-sequential property of the two algorithms, parallel 
computing based on multi-threading is usually considered helpful.
Following PA1, we use Llama3, an open-source variation of GPT for this assignment. A single-thread C 
implementation of the inference program, named seq.c, is provided as the starting point of your 
work. You need to use POSIX Pthreads with either the semaphore or (mutex_lock + condition variable) 
to implement a multi-threading version of the inference program, which parallelizes both matrix vector-multiplication and multi-head attention functions. This multi-threading version shall 
significantly accelerate the inference task of the Large Language Model. Moreover, to reduce the 
system workload in creating multiple threads, you need to reuse threads by formulating them into a 
thread pool.
Acknowledgement: The inference framework used in this assignment is based on the open-source 
project llama2.c by Andrej Karpathy. The LLM used in this assignment is based on SmolLM by 
HuggingfaceTB. Thanks open-source!
GPT-based Large Language Model
At high-level, GPT is a machine that can generate words one by one based on previous words (also 
known as prompts), and Figure 1a illustrates the basic workflow of GPT on generating “How are you”:
Figure 1. GPT Insight. a) GPT generates text one by one, and each output is the input of the next generation. b) GPT has
four major components: Tokenizer turns word (string) into a vector, Softmax + Sample gives the next token, and each layer 
has Attention and FFN (Feed-Forward Network), consisting of many Matrix-Vector-Multiplication
Figure 1b showcases the inference workflow of each word like “You” in “How are you”: First, words 
are transformed into token embeddings using the tokenizer, which is essentially a (python) 
dictionary that assigns a unique vector to each word. Embedding vectors go through multiple layers, 
which involve three steps.
• The first step is Multi-Head Attention, where the model first calculates attention scores based on 
the cosine similarity between the current word's query embedding and embeddings of previous 
words (keys). Then weighted average value embeddings are used to formulate output.
• The second step is a feed-forward network (FFN) that applies more learnable parameters 
through Matrix-Vector-Multiplication.
• The third step is positional embedding, which takes into account the ordering of words in natural 
language by adding positional information by RoPE (not required in this assignment).
After going through all the layers, the embeddings are classified to generate a specific word as the 
output. This involves using a softmax function to convert the embeddings into a probability 
distribution, and randomly sample a word from the distribution.
Task 1: Matrix-Vector-Multiplication
Figure 2. Matrix-Vector-Multiplication Algorithm.
As shown in Figure 2, Matrix-Vector-Multiplication can be illustrated as two iterations:
For Each Row i
For Column j, accumulate Matrix[i][j] * Vector[j] to Out[i]
More specifically, a sample single-thread C implementation isshown below:
void mat_vec_mul(float* out, float* vec, float* mat, int col, int row) { 
for (int i = 0; i   # prompt must quoted with "", recommend using only one prompt 
# examples, more to be found in bench.txt
./seq 42 "What’s Fibonacci Number?"
./seq 42 "Why didn't my parents invite me to their wedding?"
Upon invocation, the program will configure the random seed and begin sentence generation based 
on the provided prompt. The program calls the forward function to call LLM and generate the 
next token, and the printf() with fflush() to print the generated word to the terminal
immediately. A pair of utility time measurement functions time_in_ms will measure the time 
with millisecond accuracy.
Finally, when generation isfinished, the length, average speed, and system usage will be printed:
$ ./seq 42 "What is Fibonacci Number?" 
User: What is Fibonacci Number? 
assistant
A Fibonacci sequence is a sequence of numbers in which each number is the sum of the two 
preceding numbers (1, 1, 2, 3, 5, 8, 13, ...)
......
F(n) = F(n-1) + F(n-2) where F(n) is the nth Fibonacci number. 
length: 266, speed (tok/s): 21.4759
main thread – user: 12.4495 s, system: 0.0240 s
By fixing the same machine (workbench2) and the same random seed, generated text can be 
exactly replicated. For example, the above sample is from the test we conducted on workbench2
with random seed 42. Moreover, achieved tok/s represents the average number of tokens 
generated within a second, and we use it as the metric for speed measurement. Due to the
fluctuating system load from time to time, the speed of the generation will fluctuate around some 
level.
b. Implement the parallel mat_vec_mul and multi_head_attn by multi-threading
Open the parallel_[UID].c, rename [UID] with your UID, and open Makefile, rename [UID] 
with your UID (make sure no blank space after). Implement the thread pool with the workflow 
illustrated in Figure. 4 by completing the required five functions and adding appropriate global 
variables or more functions if needed.
For multi-threading, please use pthread.h. For synchronization, please use either semaphore or 
(mutex locks and conditional variables). You can only modify the code between specified // YOUR 
CODE STARTS HERE at line 28 and // YOUR CODE ENDS HERE at line 88.
Here are some suggestions for the implementation:
1. How to assign new parameters to threads and inform threads to terminate?
i. Via variables in globalspace. Noted that thr_func can access global variables.
ii. Via pointers to parameters. The main thread changes pointers to params before wake-up
worker threads.
2. How to assign new tasks (mat_vec_mul or multi_head_attn)? Via function pointers.
3. Once the main thread invokes mat_vec_mul() or multi_head_attn(), it should wait for all 
computations to complete before returning from the function.
make -B # applicable after renaming [UID]
gcc -o parallel parallel_[UID].c -O2 -lm -lpthread # or manually via gcc
./parallel 4 42 "What is Fibonacci Number?" 
User: What is Fibonacci Number?
assistant
A Fibonacci sequence is a sequence of numbers in which each number is the sum of the two 
preceding numbers (1, 1, 2, 3, 5, 8, 13, ...)
......
F(n) = F(n-1) + F(n-2) where F(n) is the nth Fibonacci number. 
length: 266, speed (tok/s): 38.8889 
Thread 0 has completed - user: 4.9396 s, system: 0.1620 s
Thread 1 has completed - user: 4.7195 s, system: 0.1806 s
Thread 2 has completed - user: 4.6274 s, system: 0.1843 s
Thread 3 has completed - user: 5.0763 s, system: 0.1702 s
main thread - user: 0.6361 s, system: 0.6993 s 
Whole process - user: 20.0198 s, system: 1.3757 s
4. For collecting system usage upon finish, please use getrusage.
Your implementation shall be able to be compiled by the following command:
The program accepts three arguments, (1) thr_count , (2) seed, and (3) the prompt (enclosed by 
""). Code related to reading arguments has been provided in parallel_[UID].c. You ca代 写COMP3230、Python
代做程序编程语言n use 
thr_count to specify the number of threads to use.
./parallel   
If your implementation is correct, under the same random seed, the generated text shall be the
same as the sequential version, but the generation will be faster. Moreover, you should report on
the system usage for each thread respectively (including the main thread) and the whole program. 
For example, this is the output of random seed 42 on workbench2 with 4 threads:
c. Measure the performance and report your findings
Benchmark your implementation (tok/s) on your environment (Linux or WSL or docker or Github
Codespaces) with different thread numbers and report metrics like the following table:
Thread Numbers Speed (tok/s) User Time System Time Use Time/System Time
0 (Sequential)
1
2
4
6
8
10
12
16
Regarding system usage (user time / system time), please report the usage of the whole process 
instead of each thread. Then based on the above table, try to briefly analyze the relation between 
performance and the number of threads and reason the relationship. Submit the table, your analysis, 
and your reasoning in a one-page pdf document.
IMPORTANT: Due to the large number of students this year, please conduct the benchmark on your 
own computer instead of the workbench2 server. The grading of your report is based on your 
analysis and reasoning instead of the speed you achieved. When you’re working on workbench2, 
please be reminded that you have limited maximum allowed thread numbers (128) and process 
(512), so please do not conduct benchmarking on the workbench2 server.
Submission
Submit your program to the Programming # 2 submission page on the course’s Moodle website. 
Name the program to parallel _[UID].c (replace [UID] with your HKU student number). As 
the Moodle site may not accept source code submission, you can compress files to the zip format 
before uploading. Submission checklist:
• Yoursource code parallel_[UID].c, must be self-contained with no dependencies other than
model.h and common.h) and Makefile.
• Your report must include the benchmark table, your analysis, and your reasoning.
• Your GenAI usage report contains GenAI models used (if any), prompts and responses.
• Please do not compress and submit the model and tokenizer binary files. (make clean_bin)
Documentation
1. At the head of the submitted source code, state the:
• File name
• Student’s Name and UID
• Development Platform (Please include compiler version by gcc -v)
• Remark – describe how much you have completed (See Grading Criteria)
2. Inline comments (try to be detailed so that your code can be understood by others easily)
Computer Platform to Use
For this assignment, you can develop and test your program on any Linux platform, but you must 
make sure that the program can correctly execute on the workbench2 Linux server (as the tutors
will use this platform to do the grading). Your program must be written in C and successfully 
compiled with gcc on the server.
It’s worth noticing that the only server for COMP3230 is workbench2.cs.hku.hk, and please do not 
use any CS department server, especially academy11 and academy21, as they are reserved for 
other courses. In case you cannot login to workbench2, please contact the tutor(s) for help.
Grading Criteria
1. Your submission will be primarily tested on the workbench2 server. Make sure that your 
program can be compiled without any errors. Otherwise, we have no way to test your 
submission, and you will get a zero mark.
2. As the tutor will check your source code, please write your program with good readability (i.e., 
with clear code and sufficient comments) so that you will not lose marks due to confusion.
3. You can only use pthread.h and semaphore.h (if needed), using other external libraries
like OpenMP, BLAS or LAPACK will lead to a 0 mark.
Detailed Grading Criteria
• Documentation -1 point if failed to do
• Include necessary documentation to explain the logic of the program
• Include the required student’s info at the beginning of the program
• Report: 1 point
• Measure the performance of the sequential program and your parallel program on your 
computer with various No. of threads (0, 1, 2, 4, 6, 8, 10, 12, 16).
• Briefly analyze the relation between performance and No. of threads, and reason the relation
• Implementation: 11 points evaluated progressively
1. (+3 points = 3 points) Achieve correct result  use multi-threading. Correct means 
generated text of multi-threading and sequential are identical with the same random
seed.
2. (+3 points = 6 points total) All in 1., and achieve >10% acceleration by multi-threading 
compared with sequential under 4 threads. Acceleration measurement is based on tok/s, 
acceleration must result from multi-threading instead of others like compiler (-O3), etc.
3. (+2 points = 8 points total) All in 2., and reuse threads in multi-threading. Reuse threads 
means the number of threads created in the whole program must be constant as 
thr_count.
4. (+3 points = 11 points total) All in 3., and mat_vec_mul and multi_head_attn use the same 
thread pool. Reusing the same thread pool means there’s only one pool and one thread 
group.
Plagiarism
Plagiarism is a very serious offense. Students should understand what constitutes plagiarism, the 
consequences of committing an offense of plagiarism, and how to avoid it. Please note that we may 
request you to explain to us how your program is functioning as well as we may also make use of 
software tools to detect software plagiarism.
GenAI Usage Guide
Following course syllabus, you are allowed to use Generative AI to help completing the assignment, 
and please clearly state the GenAI usage in GenAI Report, including:
• Which GenAI models you used
• Your conversations, including your prompts and the responses
Appendix
a. Parallelism Checking
To parallel by multi-threading, it’s critical to verify if the computation is independent to avoid race 
conditions and the potential use of lock. More specifically, we need to pay special attention to
check and avoid writing to the same memory location while persisting the correctness.
For example, the 1st iteration (outer for-loop) matches the requirement of independence as the 
computation of each row won’t affect others, and the only two writing is out[i] and val. Writing to 
the same out[i] can be avoided by separating i between threads. val can be implemented as stack 
variables for each thread respectively so no writing to the same memory.
Quite the opposite, 2nd iteration (inner for-loop) is not a good example for multi-threading, though 
the only writing is val. If val is implemented as a stack variable, then each thread only holds a part
of the correct answer. If val is implemented as heap variables to be shared among threads, then 
val requires a lock for every writing to avoid race writing, which is obviously costly.
b. Design of Context
A straightforward solution to the above problem is to let thr_func (thread function) do computation
and exit when finished and let original mat_vec_mul or multi_head_attn to create threads and wait 
for threads to exit by pthread_join. This could provide the same synchronization.
However, this implementation is problematic because each function call to mat_vec_mul or 
multi_head_attn will create 𝑛 new threads. Unfortunately, to generate a sentence, GPTs will call 
mat_vec_mul or multi_head_attn thousands of times, so thousands of threads will be created and 
destroyed, which leads to significant overhead to the operation system.
Noted that all the calls to mat_vec_mul function or multi_head_attn are doing the same task, i.e., 
Matrix-Vector-Multiplication, and the only difference between each function call is the parameters. 
Thus, a straightforward optimization is to reuse the threads. In high-level, we can create N threads in 
advance, and when mat_vec_mul or multi_head_attn is called, we assign new parameters for 
thread functions and let threads work on new parameters.
Moreover, it’s worth noting that mat_vec_mul and multi_head_attn are only valid within the 
context, i.e., between init_thr_pool and close_thr_pool, or there are no threads other than the 
main (not yet created or has been destroyed). This kind of context provides efficient and robust 
control over local variables and has been integrated with high-level languages like Python.

         
加QQ：99515681  WX：codinghelp  Email: 99515681@qq.com
