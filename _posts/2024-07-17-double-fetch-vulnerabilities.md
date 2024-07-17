---
layout: post
title: double fetch vulnerabilities
tags: [security]
---

I recently read [double fetch vulnerabilities](https://research.nccgroup.com/wp-content/uploads/2022/03/NCC_Group_Whitepaper_DoubleFetch2022_Report_2022-03-25_v1.0.pdf). In the process, I was confused by how the vulnerabilities are exploited. With learning about some basic knowledge and thoughts, I have finally comprehended it and written my thoughts about double fetch vulnerabilities.
## Vulnerabilities
To put it simply, `double fetch vulnerabilities` occurs when one variant is used to determine a key value or action at first, and the variant is used again to determine another key value or action assuming that the value of the variant on the second fetch is the same as the one on the first fetch.
### Programming Practices Causing Double Fetch
```c
// Attacker controls lParam
void win32k_entry_point(...){
    [...]
    // lParam has already passed successfully the ProbeForRead
    my_struct = (PMY_STRUCT)lParam;
    if (my_struct ->lpData) {
        cbCapture = sizeof(MY_STRUCT) + my_struct->cbData; // [1] first fetch
        [...]
        // my_struct ->lpData has already passed successfully the ProbeForRead
        [...]
        my_allocation = UserAllocPoolWithQuota(cbCapture, TAG_SMS_CAPTURE);
        if ( my_allocation != NULL) {
            RtlCopyMemory(my_allocation, my_struct->lpData,
            my_struct->cbData); // [2] second fetch
        }
    }
    [...]
}
```
In this function, the attacker can control the `cbData` within `my_struct`. At first, this was confusing because I thought the entire `my_struct` was owned by the function. However, I realized there are multiple ways the attacker can modify the `cbData` variant within `my_struct` before the second fetch especially because it is intialized with parameters controlled by the attacker.

If the attacker is able to control the cbData before the second fetch, then the size of `my_allocation` will no longer match the actual size of the data pointed to by `lpData` (which is now based on the new, attacker-controlled cbData). This mismatch can lead to a buffer overflow vulnerability, as the function will attempt to copy more data into my_allocation than it can safely hold. This discrepancy between the allocated buffer size and the actual data size is what causes the potential overflow error.
### Compiler-Introduced Double Fetch
Here is the vulnerable code.
```c
#include <stdio.h>
#include <stdlib.h>
void doStuff(int* ps){
    printf("NON-VOLATILE");
    switch(*ps){
        case 0: { printf("0"); break; }
        case 1: { printf("1"); break; }
        case 2: { printf("2"); break; }
        case 3: { printf("3"); break; }
        case 4: { printf("4"); break; }
        default: { printf("default"); break; }
    }
    return;
}
void main(int argc, void *argv){
    int i = rand();
    doStuff(&i);
}
```
To demonstate the issue, I use `x86-64 gcc 14.1` to generate parts of its assembly code.
```
doStuff:
    push    rbx
    xor     eax, eax
    mov     rbx, rdi
    mov     edi, OFFSET FLAT:.LC0
    call    printf
    cmp     DWORD PTR [rbx], 4 # first fetch
    ja      .L2
    mov     eax, DWORD PTR [rbx] # second fetch
    jmp     [QWORD PTR .L4[0+rax*8]]
    ...
```
With two or more threads executing this assembly code, there may be a race condition. In this case, each thread tries to access the shared memory location `[rbx]` and potentially modify its value. As a result, the value fetched on the second read may not be the same as the first read. This makes the destination of the final jmp instruction, potentially leading to arbitrary code execution.