# Description for LCI v0.10.5 Null Pointer Dereference

LCI version 0.10.5 was discovered to contain a null pointer dereference in the function nextToken()

## Reproduction

Download and compile the LCI interpreter:

```
git clone https://github.com/justinmeza/lci.git
cd lci
cmake .
make
```

To reproduce the issue, execute LCI against the attached progam named **nullderef.lol**.   Notice the interpreter segfaults immediatlely as the interpreter attempts to dereference the value stored in the RAX register, which is null (0). 

```
$ ./lci nullderef.lo

segmentation fault  ./lci nullderef.lol
```

Further debugging with ASAN gives better clarity regarding the exact location of the null pointer dereference, and helped confirm the existence of the vulnerability.  To compile with ASAN, you can add the following configuration options to the CMakeLists.txt file when building LCI from source:

```
add_compile_options(-fsanitize=address)
add_link_options(-fsanitize=address)
```

Executing the program produces the following result from ASAN:
```
AddressSanitizer:DEADLYSIGNAL
=================================================================
==1077686==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x5597acc7b2c2 bp 0x7fffd09715f0 sp 0x7fffd09715d0 T0)
==1077686==The signal is caused by a READ memory access.
==1077686==Hint: address points to the zero page.
    #0 0x5597acc7b2c2 in nextToken (/home/kali/projects/fuzzing/lci/lci+0x21f2c2)
    #1 0x5597acc80b11 in parseLoopStmtNode (/home/kali/projects/fuzzing/lci/lci+0x224b11)
    #2 0x5597acc8274b in parseStmtNode (/home/kali/projects/fuzzing/lci/lci+0x22674b)
    #3 0x5597acc82a38 in parseBlockNode (/home/kali/projects/fuzzing/lci/lci+0x226a38)
    #4 0x5597acc81c0c in parseFuncDefStmtNode (/home/kali/projects/fuzzing/lci/lci+0x225c0c)
    #5 0x5597acc82778 in parseStmtNode (/home/kali/projects/fuzzing/lci/lci+0x226778)
    #6 0x5597acc82a38 in parseBlockNode (/home/kali/projects/fuzzing/lci/lci+0x226a38)
    #7 0x5597acc81c0c in parseFuncDefStmtNode (/home/kali/projects/fuzzing/lci/lci+0x225c0c)
    #8 0x5597acc82778 in parseStmtNode (/home/kali/projects/fuzzing/lci/lci+0x226778)
    #9 0x5597acc82a38 in parseBlockNode (/home/kali/projects/fuzzing/lci/lci+0x226a38)
    #10 0x5597acc82d7f in parseMainNode (/home/kali/projects/fuzzing/lci/lci+0x226d7f)
    #11 0x5597acc77dc9 in main (/home/kali/projects/fuzzing/lci/lci+0x21bdc9)
    #12 0x7f8baecba209 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #13 0x7f8baecba2bb in __libc_start_main_impl ../csu/libc-start.c:389
    #14 0x5597acc69350 in _start (/home/kali/projects/fuzzing/lci/lci+0x20d350)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: SEGV (/home/kali/projects/fuzzing/lci/lci+0x21f2c2) in nextToken
==1077686==ABORTING   
```

Reviewing the backtrace with GDB shows the location of the following comparison in the function nextToken() that results in a null pointer dereference:


**GDB Backtrace**
```
Program received signal SIGSEGV, Segmentation fault.
0x00005555555e2c41 in nextToken ()
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
────────────────────────────────────────────────────────────────────────────────────────────────[ REGISTERS ]─────────────────────────────────────────────────────────────────────────────────────────────────
 RAX  0x0
 RBX  0x0
 RCX  0x26
 RDX  0x5555556d9f48 —▸ 0x5555556d9e20 ◂— 0xd /* '\r' */
 RDI  0x7fffffffd938 —▸ 0x5555556d9f48 —▸ 0x5555556d9e20 ◂— 0xd /* '\r' */
 RSI  0x42
 R8   0x3
 R9   0x5555556d7e90 ◂— 'vuln/nullderef.lo'
 R10  0x0
 R11  0x7ffff7ebbc60 (main_arena) ◂— 0x0
 R12  0x7fffffffde98 —▸ 0x7fffffffe215 ◂— '/dev/shm/lci/lci'
 R13  0x5555555e100b (main) ◂— push   rbp
 R14  0x5555556aade0 (__do_global_dtors_aux_fini_array_entry) —▸ 0x5555555da2e0 (__do_global_dtors_aux) ◂— endbr64 
 R15  0x7ffff7ffd020 (_rtld_global) —▸ 0x7ffff7ffe2c0 —▸ 0x555555554000 ◂— 0x10102464c457f
 RBP  0x7fffffffd910 —▸ 0x7fffffffda10 —▸ 0x7fffffffda60 —▸ 0x7fffffffdab0 —▸ 0x7fffffffdb20 ◂— ...
 RSP  0x7fffffffd910 —▸ 0x7fffffffda10 —▸ 0x7fffffffda60 —▸ 0x7fffffffdab0 —▸ 0x7fffffffdb20 ◂— ...
 RIP  0x5555555e2c41 (nextToken+33) ◂— mov    eax, dword ptr [rax]
──────────────────────────────────────────────────────────────────────────────────────────────────[ DISASM ]──────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0x5555555e2c41 <nextToken+33>       mov    eax, dword ptr [rax]
   0x5555555e2c43 <nextToken+35>       cmp    dword ptr [rbp - 0x1c], eax
   0x5555555e2c46 <nextToken+38>       je     nextToken+47                <nextToken+47>
    ↓
   0x5555555e2c4f <nextToken+47>       mov    eax, 1
   0x5555555e2c54 <nextToken+52>       pop    rbp
   0x5555555e2c55 <nextToken+53>       ret    
 
   0x5555555e2c56 <parser_error>       push   rbp
   0x5555555e2c57 <parser_error+1>     mov    rbp, rsp
   0x5555555e2c5a <parser_error+4>     sub    rsp, 0x10
   0x5555555e2c5e <parser_error+8>     mov    dword ptr [rbp - 4], edi
   0x5555555e2c61 <parser_error+11>    mov    qword ptr [rbp - 0x10], rsi
──────────────────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────────────────
00:0000│ rbp rsp 0x7fffffffd910 —▸ 0x7fffffffda10 —▸ 0x7fffffffda60 —▸ 0x7fffffffdab0 —▸ 0x7fffffffdb20 ◂— ...
01:0008│         0x7fffffffd918 —▸ 0x5555555e58dd (parseLoopStmtNode+1023) ◂— test   eax, eax
02:0010│         0x7fffffffd920 —▸ 0x7ffff7ffd020 (_rtld_global) —▸ 0x7ffff7ffe2c0 —▸ 0x555555554000 ◂— 0x10102464c457f
03:0018│         0x7fffffffd928 —▸ 0x7fffffffda88 —▸ 0x5555556d9f38 —▸ 0x5555556d9ca0 ◂— 0x3a /* ':' */
04:0020│         0x7fffffffd930 ◂— 0x0
05:0028│ rdi     0x7fffffffd938 —▸ 0x5555556d9f48 —▸ 0x5555556d9e20 ◂— 0xd /* '\r' */
06:0030│         0x7fffffffd940 ◂— 0x0
07:0038│         0x7fffffffd948 ◂— 0x1ffffd9a0
────────────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────────────────────────────────────
 ► f 0   0x5555555e2c41 nextToken+33
   f 1   0x5555555e58dd parseLoopStmtNode+1023
   f 2   0x5555555e6a3a parseStmtNode+825
   f 3   0x5555555e6b90 parseBlockNode+81
   f 4   0x5555555e6381 parseFuncDefStmtNode+443
   f 5   0x5555555e6a64 parseStmtNode+867
   f 6   0x5555555e6b90 parseBlockNode+81
   f 7   0x5555555e6381 parseFuncDefStmtNode+443


```


## References

* [OWASP Null Pointer Dereference](https://owasp.org/www-community/vulnerabilities/Null_Dereference)

