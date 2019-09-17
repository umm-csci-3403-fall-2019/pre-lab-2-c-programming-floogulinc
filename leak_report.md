# Leak report

Running `valgrind` on the compiled program gives:
```
floogulinc@floogulinc-T480s:/mnt/c/Users/floogulinc/Documents/Code/Git Repositories/pre-lab-2-c-amming-floogulinc$ valgrind --leak-check=full ./check_whitespace
==335== Memcheck, a memory error detector
==335== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==335== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==335== Command: ./check_whitespace
==335==
==335== error calling PR_SET_PTRACER, vgdb might block
The string 'Morris' is clean.
The string '  stuff' is NOT clean.
The string 'Minnesota' is clean.
The string 'nonsense  ' is NOT clean.
The string 'USA' is clean.
The string '   ' is NOT clean.
The string '     silliness    ' is NOT clean.
==335==
==335== HEAP SUMMARY:
==335==     in use at exit: 46 bytes in 6 blocks
==335==   total heap usage: 7 allocs, 1 frees, 4,142 bytes allocated
==335==
==335== 46 bytes in 6 blocks are definitely lost in loss record 1 of 1
==335==    at 0x4C31B25: calloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==335==    by 0x10882E: strip (check_whitespace.c:41)
==335==    by 0x10889A: is_clean (check_whitespace.c:62)
==335==    by 0x108946: main (check_whitespace.c:87)
==335==
==335== LEAK SUMMARY:
==335==    definitely lost: 46 bytes in 6 blocks
==335==    indirectly lost: 0 bytes in 0 blocks
==335==      possibly lost: 0 bytes in 0 blocks
==335==    still reachable: 0 bytes in 0 blocks
==335==         suppressed: 0 bytes in 0 blocks
==335==
==335== For counts of detected and suppressed errors, rerun with: -v
==335== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

Looking at this output, It seems the `calloc` in the `strip` function is the issue.

```c
  // Allocate a slot for all the "saved" characters
  // plus one extra for the null terminator.
  result = calloc(size-num_spaces+1, sizeof(char));
  // Copy in the "saved" characters.
  for (i=first_non_space; i<=last_non_space; ++i) {
    result[i-first_non_space] = str[i];
  }
  // Place the null terminator at the end of the result string.
  result[i-first_non_space] = '\0';

  return result;
 ```

The fix is to free `result` at some point but this can't be done within the `strip` function as it would be unable to return `result`.

The solution would be to free it after it is returned and set to a `cleaned` in `is_cleaned` with the condition that if the length of `cleaned` is 0, it doesn't need to be freed and attempting to do so will result in an `invalid pointer` error. So there neeeds to be a check for the length of `cleaned`.

```c
/*
 * Return true (1) if the given string is "clean", i.e., has
 * no spaces at the front or the back of the string.
 */
int is_clean(char* str) {
  char* cleaned;
  int result;

  // We check if it's clean by calling strip and seeing if the
  // result is the same as the original string.
  cleaned = strip(str);

  // strcmp compares two strings, returning a negative value if
  // the first is less than the second (in alphabetical order),
  // 0 if they're equal, and a positive value if the first is
  // greater than the second.
  result = strcmp(str, cleaned);

  // Free cleaned after it has been checked if it is of length > 0
  if (strlen(cleaned) != 0) {
    free(cleaned); 
  }

  return result == 0;
}
```

This change results in the following output from `valgrind`:
```
==348== HEAP SUMMARY:
==348==     in use at exit: 0 bytes in 0 blocks
==348==   total heap usage: 7 allocs, 7 frees, 4,142 bytes allocated
==348==
==348== All heap blocks were freed -- no leaks are possible
==348==
==348== For counts of detected and suppressed errors, rerun with: -v
==348== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```