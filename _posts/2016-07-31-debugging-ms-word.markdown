---
layout: post
title:  "Debugging MS Word with WinDbg"
date:   2016-07-22 04:46:01 +0500
tags: ["windbg", "debugging"]
---

### The issue
One of the more exotic things we've had to deal with recently was debugging MS Word hanging in production.
We use Word via COM Interop (it's [not supported][word] and very ugly. We're trying to move away from it) to convert word documents to various formats. The service responsible for printing runs a pool of Word instances and uses one of them when needed. 

At some point a replica of the service  exhausted all instances of MS Word in its' pool by requests that acquired an instance from the pool, invoked into Word and never completed. All requests coming after that would wait to acquire an instance of Word indefinitely. On the front end server side, the request to the print service would eventually time out, but the print service thread would continue waiting.

An obvious bug was that we did not have timeouts for operations calling into MS Word and no mechanism to force a request to go to the next replica if a replica is reachable but not healthy. So we went to fix that and it reduced the number of errors our clients saw. But a single replica still continued to fail by exhausting its' pool and becoming unavailable.

So we needed to figure out what was happening inside MS Word. We have a memory dump of MS Word (when dealing with something that happened in production get a memory dump first, ask questions later), so we can analyse that.

Using `!uniqstack` immediately gives the reason why Word is hanging: it's waiting for a modal dialog. Cool. MS warned that we shouldn't be using Office on a server for this exact reason.

```
0:008> k
 # Child-SP          RetAddr           Call Site
00 00000000`0676f708 00000000`776f4bc4 user32!NtUserWaitMessage+0xa
01 00000000`0676f710 00000000`776f4edd user32!DialogBox2+0x274
02 00000000`0676f7a0 00000000`77742920 user32!InternalDialogBox+0x135
03 00000000`0676f800 00000000`77741c15 user32!SoftModalMessageBox+0x9b4
04 00000000`0676f930 00000000`7774146b user32!MessageBoxWorker+0x31d
05 00000000`0676faf0 00000000`77741362 user32!MessageBoxTimeoutW+0xb3
06 00000000`0676fbc0 000007fe`e3e5ff93 user32!MessageBoxW+0x4e
07 00000000`0676fc00 00000000`775c59ed WWLIB!GetAllocCounters+0xafe8bf
08 00000000`0676fc40 00000000`777fc541 kernel32!BaseThreadInitThunk+0xd
09 00000000`0676fc70 00000000`00000000 ntdll!RtlUserThreadStart+0x1d
```

The signature of MessageBoxW is: 
```C++
int WINAPI MessageBox(
  _In_opt_ HWND    hWnd,
  _In_opt_ LPCTSTR lpText,
  _In_opt_ LPCTSTR lpCaption,
  _In_     UINT    uType);``` It seems that it could be useful to know what the message box is showing, so we need to find the values of parameters lpText and lpCaption. `kp` seems to be able to show parameter values for a frame, but it does not work without private symbols. Too bad. `kb` can show the first three arguments. Let's try that.  

```
0:008> kb
 # RetAddr           : Args to Child                                                           : Call Site
...
06 000007fe`e3e5ff93 : 00000000`00236c10 00000000`00000000 00000000`00000000 00000000`00000030 : user32!MessageBoxW+0x4e
07 00000000`775c59ed : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : WWLIB!GetAllocCounters+0xafe8bf
08 00000000`777fc541 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0xd
09 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x1d
```

This doesn't make sense because it seems to show that MessageBoxW was called with null strings. But we tried to plug some of the values we saw in args into the 'Memory' window of WinDbg, and saw some  interesting strings _near_ one of the addresses above.

```
00000000`00236d12 W o r d   c o u l d   n o t   c r e a t e   t h e  
00000000`00236d46 w o r k   f i l e .   C h e c k   t h e   t e m p  
00000000`00236d7a e n v i r o n m e n t   v a r i a b l e . . . . .
...
00000000`00236e7e . . C : \ U s e r s \ u s e r \ A p p D a t a \ L o
00000000`00236eb2 c a l \ M i c r o s o f t \ W i n d o w s \ T e m p
00000000`00236ee6 o r a r y   I n t e r n e t   F i l e s \ C o n t e
00000000`00236f1a n t . W o r d \ ~ W R S { 7 9 A C F D C 1 - F E F 4
00000000`00236f4e - 4 A 9 C - B 3 1 0 - C 2 C 9 5 F 9 6 6 2 9 C } . t
00000000`00236f82 m p . . . . . . . . . . . . . . . . . . . . . . 
```

I realise that being _near_ an address is not scientific.
But at this point we jumped to the conclusion that these strings are too specific to be just generic strings stored in resources 'just in case' so must have been generated at some point during the process's lifetime and must have been _the_ values shown in the message box. 

We confirmed with procexp that a working installation of MS Word _does_ actively use the Content.Word directory for temporary files, looked at the directory on the misbehaving machine and saw that it had incorrect permissions that prevented us from writing to it with normal Windows means. (Why did Word fail only intermittently and how did it manage to work the rest of the time?). Tried to reproduce this on a different machine by removing all permissions to the folder and got the expected result - Word hung at startup.

So we reset permissions on the folder, and the issue has not reoccured.

Fixed!

### We need to go deeper
Some time later I thought it could be interesting to figure out where we took a wrong turn in our analysis, since it seemed like we stumbled onto the cause of the issue completely by accident. Time to get out our crash dump. 

Turns out Windows uses different calling conventions on x86 and x64. WinApi x86 uses _stdcall, which means all function parameters are pushed onto the stack, so parameters of all functions above the current frame stay on the stack (where they can be viewed). On x64, the [calling convention][x64fastcall] is a variation of fastcall, so the first four parameters are passed using the registers rcx, rdx, r8 and r9. In non-optimised builds, the compiler generates code to store these registers on the stack in 20h bytes of 'home space' which must be allocated by a function's caller. With optimisation on, it does whatever it wants to do. [This article][debugging] (see the section "The Nightmare Begins") describes some debugging techniques.

Let's try to find the values passed to MessageBoxW. First, dump its' code.
<pre>
0:008> uf user32!MessageBoxW
user32!MessageBoxW:
00000000`77741314 4883ec38        sub     rsp,38h
00000000`77741318 4533db          xor     r11d,r11d
00000000`7774131b 44391d1a0e0200  cmp     dword ptr [user32!gfEMIEnable (00000000`7776213c)],r11d
00000000`77741322 742e            je      user32!MessageBoxW+0x3e (00000000`77741352)  Branch

user32!MessageBoxW+0x10:
00000000`77741324 65488b042530000000 mov   rax,qword ptr gs:[30h]
00000000`7774132d 4c8b5048        mov     r10,qword ptr [rax+48h]
00000000`77741331 33c0            xor     eax,eax
00000000`77741333 f04c0fb115ec240200 lock cmpxchg qword ptr [user32!gdwEMIThreadID (00000000`77763828)],r10
00000000`7774133c 4c8b15dd240200  mov     r10,qword ptr [user32!gpReturnAddr (00000000`77763820)]
00000000`77741343 418d4301        lea     eax,[r11+1]
00000000`77741347 4c0f44d0        cmove   r10,rax
00000000`7774134b 4c8915ce240200  mov     qword ptr [user32!gpReturnAddr (00000000`77763820)],r10

user32!MessageBoxW+0x3e:
00000000`77741352 834c2428ff      <mark>or      dword ptr [rsp+28h],0FFFFFFFFh</mark>
00000000`77741357 6644895c2420    <mark>mov     word ptr [rsp+20h],r11w</mark>
00000000`7774135d e856000000      <mark>call    user32!MessageBoxTimeoutW (00000000`777413b8)</mark>
00000000`77741362 4883c438        add     rsp,38h
00000000`77741366 c3              ret
</pre>

Nothing interesting happening here to rdx or r8. Just proxying the call to `user32!MessageBoxTimeoutW` with two additional arguments (they are in [rsp+20h] and [rsp+28h] because 20h bytes are reserved for home space). Moving along.

<pre>
0:008> uf user32!MessageBoxTimeoutW
user32!MessageBoxTimeoutW:
00000000`777413b8 48895c2408      mov     qword ptr [rsp+8],rbx
00000000`777413bd 48896c2410      mov     qword ptr [rsp+10h],rbp
00000000`777413c2 4889742418      mov     qword ptr [rsp+18h],rsi
00000000`777413c7 57              push    rdi
00000000`777413c8 4881ecc0000000  sub     rsp,0C0h
00000000`777413cf 488bd9          mov     rbx,rcx
00000000`777413d2 498bf0          <mark>mov     rsi,r8 </mark>
00000000`777413d5 488bfa          <mark>mov     rdi,rdx </mark>
00000000`777413d8 488d4c2420      lea     rcx,[rsp+20h]
00000000`777413dd 33d2            xor     edx,edx
00000000`777413df 41b898000000    mov     r8d,98h
00000000`777413e5 418be9          mov     ebp,r9d
00000000`777413e8 e8ab83faff      call    user32!memset (00000000`776e9798)
00000000`777413ed 0fb78424f0000000 movzx   eax,word ptr [rsp+0F0h]
00000000`777413f5 488364243000    and     qword ptr [rsp+30h],0
00000000`777413fb 833d3a0d020000  cmp     dword ptr [user32!gfEMIEnable (00000000`7776213c)],0
00000000`77741402 6689442478      mov     word ptr [rsp+78h],ax
00000000`77741407 8b8424f8000000  mov     eax,dword ptr [rsp+0F8h]
00000000`7774140e c744242050000000 mov     dword ptr [rsp+20h],50h
00000000`77741416 48895c2428      mov     qword ptr [rsp+28h],rbx
00000000`7774141b 48897c2438      <mark>mov     qword ptr [rsp+38h],rdi</mark>
00000000`77741420 4889742440      <mark>mov     qword ptr [rsp+40h],rsi</mark>
00000000`77741425 896c2448        mov     dword ptr [rsp+48h],ebp
00000000`77741429 8984249c000000  mov     dword ptr [rsp+9Ch],eax
...
user32!MessageBoxTimeoutW+0xa9:
00000000`77741461 488d4c2420      lea     rcx,[rsp+20h]
00000000`77741466 e88d040000      call    user32!MessageBoxWorker (00000000`777418f8)
00000000`7774146b 4c8d9c24c0000000 lea     r11,[rsp+0C0h]
...
00000000`77741483 c3              ret
</pre>

Okay, so here rdx and r8 are copied to registers and then passed on to `user32!MessageBoxWorker`. Fortunately, this time they are passed as rsp+38h and rsp+40h. The stacktrace from windbg gives us Child-SP, the value of SP before a frame (MessageBoxTimeoutW) transfers control to its' child (MessageBoxWorker). We can inspect the value of [rsp+38h] and [rsp+40h].  

<pre>
0:008> dq 00000000`0676faf0
00000000`0676faf0  01d1e926`00000000 00000000`00000000
00000000`0676fb00  00000000`00000030 000007fe`e46542f0
00000000`0676fb10  00000000`00000050 00000000`00000000
00000000`0676fb20  00000000`00000000 <mark>00000000`00236d12</mark>
00000000`0676fb30  <mark>000007fe`e46542f0</mark> 00000000`00000030

0:008> du (poi(00000000`0676faf0+38h)) 
000007fe`e46542f0  "Microsoft Word
0:008> du (poi(00000000`0676faf0+40h))
00000000`00236d12  "Word could not create the work file. Check the temp environment variable."
</pre>

We can confirm that these actually are the strings shown in the message box by messing up permissions to Word's temp folder and trying to create a new document via the GUI. The path to the temp folder does not appear anywhere in the GUI, but it is Googleable given the messages from the message box, so if we did not stumble onto it accidentally, we  probably would have still been able to fix the issue.

![error](/resources/word.png)

<!--But where did the string containing the Content.Word directory come from? Looking at the stacks of other threads and their stack pointers brings us to a frame deep inside Word code whose stack range contains the string. Unfortunately, the code for that function is currently too complex for my asm-fu.???
-->
 
### Conclusion

We need to work on making our service more resilient to failures of individual machines or components

I learned something new: Using WinDbg and reading ASM code is actually not as scary as I thought. WinDbg is a useful tool. Debugging optimised x64 code without symbols is... possible, but requires luck. 

Thanks to [gusev_p](https://twitter.com/gusev_p) for telling of his experience dealing with different calling conventions and [shparun](https://twitter.com/shparun) for suggesting WinDbg as a tool.  

### Links
[x64fastcall]: https://msdn.microsoft.com/en-us/library/ms235286.aspx
[debugging]: https://blogs.msdn.microsoft.com/ntdebugging/2009/01/09/challenges-of-debugging-optimized-x64-code/
[word]: https://support.microsoft.com/en-us/kb/257757

* [Common WinDbg Commands](http://windbg.info/doc/1-common-cmds.html#7_symbols)
* [Introduction to x64 Assembly](https://software.intel.com/en-us/articles/introduction-to-x64-assembly)
