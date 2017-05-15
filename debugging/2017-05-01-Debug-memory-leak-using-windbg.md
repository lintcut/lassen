
# Debug Memory Leak Using WinDBG #

## Issue Description

When I ran a program (debug version) in Visual Studio 2015, I noticed that there a lot of memory leak warnings. Like this - 

```
Detected memory leaks!
Dumping objects ->
{62} normal block at 0x008B3E40, 5 bytes long.
    Data: <Hello> 48 65 6C 6C 6F 
{61} normal block at 0x008B0548, 20 bytes long.
    Data: < > CD CD CD CD CD CD CD CD CD CD CD CD CD CD CD CD 

...

Object dump complete.
```

And the biggest problem is that I have no idea where caused the leak and when the leak happened. So I need to get information of the whole process life cycle.

## Tools Selection

I used to diagnose memory leak in kernel mode using poolmon which is very straight forward. Unfortunately, there is no such a thing in user mode.

In user mode, there are 3 options I am familiar with:

- Visual Studio Profiler
- umdh
- WinDBG

Unfortuantely, I am using Visual Studio 2015 Community Edition which doesn't have full profiler feature. So option 1 is out.

**umdh** is a powerful tool. But it is hard to dump heap information at the very begining and very end of the process lifecycle only with this tool (Or maybe I don't know). So option 2 is out.

Just like kernel mode, WinDBG is my favorite :)

## Before Debug

### Enable stack back traces

Firstly, we need to enable stack back traces using following command:

```
gflags.exe /i <ImageName.exe> +ust
```

### Set Symbol Path

Actually this is normally ready when you first time set up the computer ;)

Add a system variable "_NT_SYMBOL_PATH":

```
_NT_SYMBOL_PATH = d:\symbols\private;srv*d:\symbols\public*https://msdl.microsoft.com/download/symbols
```

## Debug

#### Step 1: Lauch Process

Run WinDBG, and launch my program. It stopped at the very begining.

Run command `!heap -s`, and get following result

```
0:000> !heap -s


************************************************************************************************************************
                                              NT HEAP STATS BELOW
************************************************************************************************************************
NtGlobalFlag enables following debugging aids for new heaps:
    stack back traces
LFH Key                   : 0xa834a79dec2909a4
Termination on corruption : ENABLED
          Heap     Flags   Reserv  Commit  Virt   Free  List   UCR  Virt  Lock  Fast 
                            (k)     (k)    (k)     (k) length      blocks cont. heap 
-------------------------------------------------------------------------------------
000001976aa70000 08000002    1220    124   1020      3     9     1    0      3   LFH
0000019769100000 08008000      64      4     64      2     1     1    0      0      
-------------------------------------------------------------------------------------

```

We can see, it is very clean.

#### Step 2: Continue run the program, and exit from it

Run command `g` let program continue, and then close my program. WinDBG shows a lot memory leak information and finally break at `NtTerminateProcess`. Like this:

```
Detected memory leaks!
Dumping objects ->
{118722} normal block at 0x0000019F71CD4780, 16 bytes long.
 Data: <  @s            > B0 1E 40 73 9F 01 00 00 00 00 00 00 00 00 00 00 
{118721} normal block at 0x0000019F73401EA0, 64 bytes long.
 Data: <  @s    ` @s    > 20 03 40 73 9F 01 00 00 60 04 40 73 9F 01 00 00 
{118719} normal block at 0x000001976F140F50, 32 bytes long.
 Data: <`  K            > 60 E4 1B 4B F7 7F 00 00 02 00 00 00 CD CD CD CD 
{118717} normal block at 0x0000019F71CD5350, 16 bytes long.
 Data: <p @s            > 70 04 40 73 9F 01 00 00 00 00 00 00 00 00 00 00 
{118716} normal block at 0x0000019F73400460, 64 bytes long.
 Data: <  @s      @s    > A0 1E 40 73 9F 01 00 00 C0 03 40 73 9F 01 00 00 
{118714} normal block at 0x000001976F1A9130, 144 bytes long.
 Data: <6 8 2 3 7 2 3 2 > 36 00 38 00 32 00 33 00 37 00 32 00 33 00 32 00 
{118713} normal block at 0x0000019F71CD4E10, 16 bytes long.
 Data: <0&@s            > 30 26 40 73 9F 01 00 00 00 00 00 00 00 00 00 00 
{118712} normal block at 0x0000019F73402620, 56 bytes long.
 Data: <   K            > E0 E4 1B 4B F7 7F 00 00 03 00 00 00 CD CD CD CD 
{118639} normal block at 0x0000019F71CD5DD0, 16 bytes long.
 Data: <  @s            > D0 03 40 73 9F 01 00 00 00 00 00 00 00 00 00 00 
{118638} normal block at 0x0000019F734003C0, 64 bytes long.
 Data: <` @s      @s    > 60 04 40 73 9F 01 00 00 20 03 40 73 9F 01 00 00 

...

 Data: <@  j            > 40 F4 AC 6A 97 01 00 00 00 00 00 00 00 00 00 00 
{2262} normal block at 0x000001976AACF430, 56 bytes long.
 Data: <   K            > E0 E4 1B 4B F7 7F 00 00 03 00 00 00 CD CD CD CD 
Object dump complete.
ntdll!NtTerminateProcess+0x14:
00007ffd`4fd453d4 c3              ret

```

Sigh! Too bad. :(
    
#### Step 3: Analyze heap

Run command `!heap -s` again.

```
0:000> !heap -s


************************************************************************************************************************
                                              NT HEAP STATS BELOW
************************************************************************************************************************
NtGlobalFlag enables following debugging aids for new heaps:
    stack back traces
LFH Key                   : 0xa834a79dec2909a4
Termination on corruption : ENABLED
          Heap     Flags   Reserv  Commit  Virt   Free  List   UCR  Virt  Lock  Fast 
                            (k)     (k)    (k)     (k) length      blocks cont. heap 
-------------------------------------------------------------------------------------
000001976aa70000 08000002   16564   9492  16364   3684   224     5    0      6   LFH
    External fragmentation  38 % (224 free blocks)
0000019769100000 08008000      64      4     64      2     1     1    0      0      
000001976ac20000 08001002    3324   2520   3124   2346    32     3    0      0   LFH
    External fragmentation  93 % (32 free blocks)
000001976c420000 08001002     260     64     60     25     8     1    0      0   LFH
0000019f6f790000 08001002      60      8     60      5     1     1    0      0      
-------------------------------------------------------------------------------------

```

The first heap `000001976aa70000` is suspicious. 

Run another command `!heap -stat -h 000001976aa70000` to check detail.

```
0:000> !heap -stat -h 000001976aa70000
 heap @ 000001976aa70000
group-by: TOTSIZE max-display: 20
    size     #blocks     total      ( %) (percent of total busy bytes)
    74          74e   -  34f58      (16.56)
    44          b9f   -  3163c      (15.45)
    c4          23d   -  1b6b4      (8.57)
    6c          3b8   -  191a0      (7.85)
    84          1c9   -  eba4       (4.61)
    5c          249   -  d23c       (4.11)
    54          266   -  c978       (3.94)
    104c        9     -  92ac       (2.87)
    100         78    -  7800       (2.35)
    6008        1     -  6008       (1.88)
    5c80        1     -  5c80       (1.81)
    120         4d    -  56a0       (1.69)
    200         23    -  4600       (1.37)
    210         1a    -  35a0       (1.05)
    50          a5    -  3390       (1.01)
    30          10d   -  3270       (0.99)
    170         20    -  2e00       (0.90)
    2c28        1     -  2c28       (0.86)
    2400        1     -  2400       (0.70)
    460         8     -  2300       (0.68)
```

Hmmm..., a lot issues.

Let's check first item -- its size is 74

Run command `!heap -flt s 74`.

```
0:000> !heap -flt s 74
    _HEAP @ 1976aa70000
              HEAP_ENTRY Size Prev Flags            UserPtr UserSize - state
        000001976aa79db0 000a 0000  [00]   000001976aa79de0    00074 - (busy)
        000001976aa79e50 000a 000a  [00]   000001976aa79e80    00074 - (busy)
        000001976aa79f90 000a 000a  [00]   000001976aa79fc0    00074 - (busy)
        000001976aa7a0d0 000a 000a  [00]   000001976aa7a100    00074 - (busy)
        000001976aa7a530 000a 000a  [00]   000001976aa7a560    00074 - (busy)
        000001976aa7a5d0 000a 000a  [00]   000001976aa7a600    00074 - (busy)
        000001976aacf1f0 000a 000a  [00]   000001976aacf220    00074 - (busy)
        000001976aacf330 000a 000a  [00]   000001976aacf360    00074 - (busy)
        000001976aacf650 000a 000a  [00]   000001976aacf680    00074 - (busy)
        000001976aacf6f0 000a 000a  [00]   000001976aacf720    00074 - (busy)
        000001976aacf8d0 000a 000a  [00]   000001976aacf900    00074 - (busy)
        000001976aacfa10 000a 000a  [00]   000001976aacfa40    00074 - (busy)
            ... ... ... ...
        000001976dcf4500 000a 000a  [00]   000001976dcf4530    00074 - (busy)
        000001976dcf45a0 000a 000a  [00]   000001976dcf45d0    00074 - (busy)
        000001976dcf4640 000a 000a  [00]   000001976dcf4670    00074 - (busy)
        000001976dcf46e0 000a 000a  [00]   000001976dcf4710    00074 - (busy)
    _HEAP @ 19769100000
    _HEAP @ 1976ac20000
    _HEAP @ 1976c420000
    _HEAP @ 19f6f790000

```

So many records.

Let's pick one and check (I pick the last one), run command `!heap -p -a 000001976dcf4710`, and get following result:

```
0:000> !heap -p -a 000001976dcf4710
    address 000001976dcf4710 found in
    _HEAP @ 1976aa70000
              HEAP_ENTRY Size Prev Flags            UserPtr UserSize - state
        000001976dcf46e0 000a 0000  [00]   000001976dcf4710    00074 - (busy)
        7ffd4fd2f883 ntdll!RtlpCallInterceptRoutine+0x000000000000003f
        7ffd4fcd84f2 ntdll!RtlpAllocateHeapInternal+0x0000000000001142
        7ffd1c52d116 ucrtbased!heap_alloc_dbg_internal+0x00000000000001f6
        7ffd1c52cebd ucrtbased!heap_alloc_dbg+0x000000000000004d
        7ffd1c52ff7f ucrtbased!_malloc_dbg+0x000000000000002f
        7ffd0499eafd mfc140ud!operator new+0x000000000000003d
*** WARNING: Unable to verify checksum for MyApp.exe
        7ff74aefa443 MyApp!std::_Allocate+0x00000000000001a3
        7ff74af40b93 MyApp!std::allocator<std::_List_node<std::pair<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,NX::rapidjson::JsonValue * __ptr64>,void * __ptr64> >::allocate+0x0000000000000043
        7ff74af40aa2 MyApp!std::_Wrap_alloc<std::allocator<std::_List_node<std::pair<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,NX::rapidjson::JsonValue * __ptr64>,void * __ptr64> > >::allocate+0x0000000000000042
        7ff74af3d9a8 +0x0000000000000058
        7ff74af2d0fe +0x000000000000005e
        7ff74af2edbd +0x000000000000007d
        7ff74af42e1e +0x000000000000006e
        7ff74af433b0 MyApp!MyObject::set+0x00000000000001b0
        7ff74af6788d MyApp!MyParser<char>::ParseObject+0x00000000000002ad
        7ff74af67fee MyApp!MyParser<char>::ParseValue+0x000000000000009e
        7ff74af66c27 MyApp!MyParser<char>::ParseArray+0x0000000000000117
        7ff74af68000 MyApp!MyParser<char>::ParseValue+0x00000000000000b0
        7ff74af67832 MyApp!MyParser<char>::ParseObject+0x0000000000000252
        7ff74af67fee MyApp!MyParser<char>::ParseValue+0x000000000000009e
        7ff74af67832 MyApp!MyParser<char>::ParseObject+0x0000000000000252
        7ff74af67fee MyApp!MyParser<char>::ParseValue+0x000000000000009e
        7ff74af67832 MyApp!MyParser<char>::ParseObject+0x0000000000000252
        7ff74af66a73 MyApp!MyParser<char>::Parse+0x0000000000000073
        7ff74af66b02 MyApp!MyStringParser<char>::Parse+0x0000000000000052
        7ff74af4e2fe MyApp!MyDocument::LoadJsonString<char>+0x000000000000005e
        7ff74b018b7d MyApp!MyRestClient::ProjectListFiles+0x000000000000036d
        7ff74b01876b MyApp!MyRestClient::ProjectListFiles+0x000000000000018b
        7ff74af86b10 MyApp!MySession::ProjectGetFirstPageFiles+0x00000000000001b0
        7ff74aef5197 MyApp!CHomePageDlg::CreateProjectElement+0x0000000000000447
        7ff74aef7138 MyApp!CHomePageDlg::InitProjectList+0x0000000000000268
        7ff74aef985d MyApp!CHomePageDlg::UpdateHomePageData+0x00000000000004ed

```

Sigh, I suddenly realized the cause in a second.

It is a stupid mistake - I wrote the clear() function but forgot to call it at class destructor.

And then I check other memory items using the same commands, and they pint to same issue.

Fix is easy, one line code, just call my clear() function in class destructor.


## After Debug

At last, since we no longer need to debug my app, just disable stack back traces using following command:

```
gflags.exe /i <ImageName.exe> -ust
```