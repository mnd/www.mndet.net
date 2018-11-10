---
layout: default
title: GCC optimization issue
---
This post was written when I found bug first time. Issue can be reproduced for sure on GCC from 4.8.2 to 8.2.0.

Simplified failure code example
-------------------------------

    $ cat test.c
    #include <stdio.h>
    int main() {
      int i = 0, res = 0;
      int array[6] = {0,0,0,0,0,0};
      while ((array[i] != 6) && (i < 6)) {
        i++;
      }
      if (i == 6) {
        res = 1;
      }
    
      printf ("res = %d, i = %d\n", res, i);
      return 0;
    }
    $ gcc --version 
    gcc (GCC) 8.2.0
    ...
    $ gcc -O2 -Wall -Wextra test.c # No warnings!
    $ ./a.out 

If you compile this code by itself there are good chance that you'll got other result but on my machine there I got next result:

    $ ./a.out 
    res = 0, i = 196
    $

Root cause of issue
-------------------

This issue for sure happens because of some optimizations. Let's try to found specific one that cause this issue:

    $ gcc -O2 -Wall -Wextra -fdump-tree-all test.c
    $ cat test.c.037t.fre1 # Last optimization without issue

    ;; Function main (main, funcdef_no=11, decl_uid=2459, cgraph_uid=11, symbol_order=11)

    main ()
    {
      int array[6];
      int res;
      int i;
      int _1;

      <bb 2> :
      array[0] = 0;
      array[1] = 0;
      array[2] = 0;
      array[3] = 0;
      array[4] = 0;
      array[5] = 0;
      goto <bb 4>; [INV]

      <bb 3> :
      i_13 = i_2 + 1;

      <bb 4> :
      # i_2 = PHI <0(2), i_13(3)>
      _1 = array[i_2];
      if (_1 != 6)
        goto <bb 5>; [INV]
      else
        goto <bb 6>; [INV]

      <bb 5> :
      if (i_2 <= 5)
        goto <bb 3>; [INV]
      else
        goto <bb 6>; [INV]

      <bb 6> :
      if (i_2 == 6)
        goto <bb 7>; [INV]
      else
        goto <bb 8>; [INV]

      <bb 7> :

      <bb 8> :
      # res_3 = PHI <0(6), 1(7)>
      printf ("res = %d, i = %d\n", res_3, i_2);
      array ={v} {CLOBBER};
      return 0;

    }

    $ cat test.c.038t.evrp # Optimization that introduce issue

    ;; Function main (main, funcdef_no=11, decl_uid=2459, cgraph_uid=11, symbol_order=11)

    ;; 2 loops found
    ;;
    ;; Loop 0
    ;;  header 0, latch 1
    ;;  depth 0, outer -1
    ;;  nodes: 0 1 2 3 4 5 6 7 8
    ;;
    ;; Loop 1
    ;;  header 4, latch 3
    ;;  depth 1, outer 0
    ;;  nodes: 4 3 5
    ;; 2 succs { 4 }
    ;; 3 succs { 4 }
    ;; 4 succs { 5 6 }
    ;; 5 succs { 3 6 }
    ;; 6 succs { 7 8 }
    ;; 7 succs { 8 }
    ;; 8 succs { 1 }

    SSA replacement table
    N_i -> { O_1 ... O_j } means that N_i replaces O_1, ..., O_j

    i_4 -> { i_2 }
    Incremental SSA update started at block: 4
    Number of blocks in CFG: 9
    Number of blocks to update: 5 ( 56%)



    Value ranges after Early VRP:

    _1: VARYING
    i_2: [0, 5]
    res_3: [0, 1]
    i_4: [0, 5]
    i_13: [1, 6]


    Removing basic block 7
    Removing basic block 5
    Merging blocks 6 and 8
    main ()
    {
      int array[6];
      int res;
      int i;
      int _1;

      <bb 2> :
      array[0] = 0;
      array[1] = 0;
      array[2] = 0;
      array[3] = 0;
      array[4] = 0;
      array[5] = 0;
      goto <bb 4>; [INV]

      <bb 3> :
      i_13 = i_2 + 1;

      <bb 4> :
      # i_2 = PHI <0(2), i_13(3)>
      _1 = array[i_2];
      if (_1 != 6)
        goto <bb 3>; [INV]
      else
        goto <bb 5>; [INV]

      <bb 5> :
      # i_4 = PHI <i_2(4)>
      printf ("res = %d, i = %d\n", 0, i_4);
      array ={v} {CLOBBER};
      return 0;

    }

So, what we can see in this code. Optimizer know that `array` can be accessed by indexes from 0 to 5 and it know that accessing `array[6]` is undefined behavior. In basic block 4 (see at last worked code) we access `array[i_2]` and as result we can assume that `0 <= i_2 <= 5`. This means that basic block 5 has useless condition `if (i_2 <= 5)` and whole block can be deleted.

So if we look again at C code:

      int array[6] = {0,0,0,0,0,0};
      while ((array[i] != 6) // Possible undefined behavior
             && (i < 6)) {   // As result this test eliminated by Early Value Range Propogation optimization 
        i++;
      }

And if we change order of operations than everything will work fine:

      int array[6] = {0,0,0,0,0,0};
      while ((i < 6) && (array[i] != 6)) { // Both checks still will be there after all optimizations
        i++;
      }

CTF style task based on this optimization
-----------------------------------------

Suppose we have application that take 3 parameters: source string, destination string, and control character. This program will find control character in source string and copy all characters prior to control character to second string:

    $ ./prog source destination r # copy "sou" over destination string
    replace 3 bytes of second arg with bytes from first
    you new string: soutination

Code of this application:

    #include <stdio.h>
    #include <string.h>
    #define MAX_LENGTH 16

    void key() {
      printf ("you got a flag\n");
    }

    int main(int argc, char **argv) {
      /* Initialize all variables */
      int i = 0;
      char arg1[MAX_LENGTH] = {0};
      char arg2[MAX_LENGTH] = {0};
      char match = '\0'; 

      if (argc != 4) return 1;

      /* Ensure that strings will be '\0' terminated */
      strncpy(arg1, argv[1], MAX_LENGTH-1);
      strncpy(arg2, argv[2], MAX_LENGTH-1);
      match = argv[3][0];

      /* Find prefix length up to `match' but not exceed array length */
      while ((arg1[i] != match) && (i < MAX_LENGTH)) {
        i++;
      }

      printf ("replace %d bytes of second arg with bytes from first\n", i);
      memcpy (arg2, arg1, i);

      printf ("you new string: %s\n", arg2);

      /* if there was replacement and after replacement strings are different then show key */
      if (i != 0 && arg1[0] != arg2[0]) key();

      return 0;
    }

Application compiled with `gcc -O2 ./test.c -Wall -Wextra -Werror -g` flags on x86-64 machine with GCC 8.2.0. You should force application to print "you got a flag" string.

Application binary: [gcc-optimization-issue-ctf-like-prog](/assets/gcc-optimization-issue-ctf-like-prog)

Solution
--------

As we already know test `i < MAX_LENGTH` will be removed from optimized code. As result we provide almost any `i` to `memcpy (arg2, arg1, i);` command. As result we can rewrite return address on a stack providing new value through `arg1`:

    $ ./gcc-optimization-issue-ctf-like-prog 11112222`printf '\xd0\x07\x40'` nevermind 0
    replace 82 bytes of second arg with bytes from first
    you new string: 11112222?@
    you got a flag
    Segmentation fault
