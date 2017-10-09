# wireshark Dissector
[Sample dissector for wireshark v2,5](https://www.wireshark.org/docs/wsdg_html_chunked/ChDissectAdd.html)


For portability some code guidelines must be followed.

- Don't initialize global or static variables in their declaration with non-constant values.
- Don't use anonymous unions, not all compilers support them.

Example:
```c
typedef struct foo{
  guint32 foo;
  union {
    guint32 foo_l;
    guint16 foo_s;
  } u; /*  Have a name here */
} foo_t;

```

Don't use "uchar", "u_char", "ushort", "u_short", "uint", "u_int",
"ulong", "u_long" or "boolean"; they aren't defined on all platforms.


Don't use "uchar", "u_char", "ushort", "u_short", "uint", "u_int",
"ulong", "u_long" or "boolean"; they aren't defined on all platforms.

Do not use "open()", "rename()", "mkdir()", "stat()", "unlink()", "remove()",
"fopen()", "freopen()" directly.  Instead use "ws_open()", "ws_rename()",
"ws_mkdir()", "ws_stat()", "ws_unlink()", "ws_remove()", "ws_fopen()",
"ws_freopen()": these wrapper functions change the path and file name from
UTF8 to UTF16 on Windows allowing the functions to work correctly when the
path or file name contain non-ASCII characters.

Also, use ws_read(), ws_write(), ws_lseek(), ws_dup(), ws_fstat(), and
ws_fdopen(), rather than read(), write(), lseek(), dup(), fstat(), and
fdopen() on descriptors returned by ws_open().

Those functions are declared in <wsutil/file_util.h>; include that
header in any code that uses any of those routines.


When opening a file with "ws_fopen()", "ws_freopen()", or "ws_fdopen()", if
the file contains ASCII text, use "r", "w", "a", and so on as the open mode
- but if it contains binary data, use "rb", "wb", and so on.  On
Windows, if a file is opened in a text mode, writing a byte with the
value of octal 12 (newline) to the file causes two bytes, one with the
value octal 15 (carriage return) and one with the value octal 12, to be
written to the file, and causes bytes with the value octal 15 to be
discarded when reading the file (to translate between C's UNIX-style
lines that end with newline and Windows' DEC-style lines that end with
carriage return/line feed).

Don't use

    case N ... M:

as that's not supported by all compilers.

Do not use functions such as strcat() or strcpy().
A lot of work has been done to remove the existing calls to these functions and
we do not want any new callers of these functions.

Instead use g_snprintf() since that function will if used correctly prevent
buffer overflows for large strings.

- - -

If you write a routine that will create and return a pointer to a filled in
string and if that buffer will not be further processed or appended to after
the routine returns (except being added to the proto tree),
do not preallocate the buffer to fill in and pass as a parameter instead
pass a pointer to a pointer to the function and return a pointer to a
wmem-allocated buffer that will be automatically freed. (see README.wmem)

I.e. do not write code such as
  static void
  foo_to_str(char *string, ... ){
     <fill in string>
  }
  ...
     char buffer[1024];
     ...
     foo_to_str(buffer, ...
     proto_tree_add_string(... buffer ...

instead write the code as
  static void
  foo_to_str(char **buffer, ...
    #define MAX_BUFFER x
    *buffer=wmem_alloc(wmem_packet_scope(), MAX_BUFFER);
    <fill in *buffer>
  }
  ...
    char *buffer;
    ...
    foo_to_str(&buffer, ...
    proto_tree_add_string(... *buffer ...

Use wmem_ allocated buffers. They are very fast and nice. These buffers are all
automatically free()d when the dissection of the current packet ends so you
don't have to worry about free()ing them explicitly in order to not leak memory.
Please read README.wmem.

## Recomendations for robustness


If there is a case where you are checking not for an invalid data item
in the packet, but for a bug in the dissector (for example, an
assumption being made at a particular point in the code about the
internal state of the dissector), use the DISSECTOR_ASSERT macro for
that purpose; this will put into the protocol tree an indication that
the dissector has a bug in it, and will not crash the application.

If you are allocating a chunk of memory to contain data from a packet,
or to contain information derived from data in a packet, and the size of
the chunk of memory is derived from a size field in the packet, make
sure all the data is present in the packet before allocating the buffer.
Doing so means that:

    1) Wireshark won't leak that chunk of memory if an attempt to
       fetch data not present in the packet throws an exception.

and

    2) it won't crash trying to allocate an absurdly-large chunk of
       memory if the size field has a bogus large value.


## Name convention

Wireshark uses the unserscore_nameconvention for function and variable names.

In order to maintain the code format use this modeline generator:
[https://www.wireshark.org/tools/modelines.html](https://www.wireshark.org/tools/modelines.html)


### Example:
Place this at the beggining of the file:
```c
/* c-basic-offset: 2; tab-width: 2; indent-tabs-mode: nil
 * vi: set shiftwidth=2 tabstop=2 expandtab:
 * :indentSize=2:tabSize=2:noTabs=true:
 */
```

Place this at the end of the file:
```c
/*
 * Editor modelines  -  https://www.wireshark.org/tools/modelines.html
 *
 * Local variables:
 * c-basic-offset: 2
 * tab-width: 2
 * indent-tabs-mode: nil
 * End:
 *
 * vi: set shiftwidth=2 tabstop=2 expandtab:
 * :indentSize=2:tabSize=2:noTabs=true:
 */
```

## From `READMe.heuristics`

When a packet is received wireshark tries to find the right dissector to start
decoding, often this can be done usign known conventions. Unfortunately these
conventions are not always followed and nothing prevents for break them.

To solve this problem, wireshark introduced the so called heuristics dissector (HD)
mechanism to try to deal with these problems.


Normal dissector are called first, if they fail into identify the packet, ie, the
data majes no sense for the dissector; then WS has a list of dissectors which register
themselves to specific type of traffic, for example TCP. From that list WS call each
one of them until one is successful in decodign the package.

Here's a sample HD working:

A HD looks into the first few packet bytes and searches for common patterns that
are specific to the protocol in question. Most protocols starts with a
specific header, so a specific pattern may look like (synthetic example):

1) first byte must be 0x42
2) second byte is a type field and can only contain values between 0x20 - 0x33
3) third byte is a flag field, where the lower 4 bits always contain the value 0
4) fourth and fifth bytes contain a 16 bit length field, where the value can't
   be larger than 10000 bytes

So the heuristic dissector will check incoming packet data for all of the
4 above conditions, and only if all of the four conditions are true there is a
good chance that the packet really contains the expected protocol - and the
dissector continues to decode the packet data. If one condition fails, it's
very certainly not the protocol in question and the dissector returns to WS
immediately "this is not my protocol" - maybe some other heuristic dissector
is interested!


if one dissector turns to be successful, will be called again for the same type of
message.



In the file `packet-udp-nm.c` is an already integrated dissector for the Udp_mn
protocol ( [info](https://github.com/yhoazk/automo/tree/master/network_protocols/udp_nm))
