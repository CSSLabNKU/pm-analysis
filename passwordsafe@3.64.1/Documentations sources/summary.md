## Cryptographic Algorithm

[Password Safe](https://appdb.winehq.org/appview.php?iAppId=4017)

![twofish](fig/twofish.png)

## Data Field

Password Safe data format

```
PasswordSafe database format description version 3.29
-----------------------------------------------------

Copyright (c) 2003-2013 Rony Shapiro <ronys@users.sourceforge.net>.
All rights reserved. Use of the code is allowed under the Artistic
License terms, as specified in the LICENSE file distributed with this
code, or available from
http://www.opensource.org/licenses/artistic-license-2.0.php 


1. Introduction: This document defines a file format for the secure
storage of passwords and related data. The format is designed
according to current cryptographic best practices, and is believed to
be secure, in the sense that without knowledge of the master
passphrase, only a brute-force attack or a flaw in the underlying
cryptographic algorithm will result in unauthorized access to the
data.

1.1 Design Goals: The PasswordSafe database format is designed to be
secure, extensible and platform-independent.

1.2 History: This specification is an evolution of previous
formats. The main differences between version 3 of the format and
previous versions are:
1.2.1. This version addresses a minor design flaw in previous versions
of the PasswordSafe database format.
1.2.3. This version replaces the underlying cryptographic functions
with more advanced versions.
1.2.4. This version allows the detection of a truncated or
corrupted/tampered database.

Meeting these goals is impossible without breaking compatibility: This
format is NOT compatible with previous (major) versions. Note,
however, that since the data stored in previous versions is a proper
subset of the data described here, implementers may read a database
written in an older version and store the result in the format
described here.

2. Format: A V3 format PasswordSafe is structured as follows:

    TAG|SALT|ITER|H(P')|B1|B2|B3|B4|IV|HDR|R1|R2|...|Rn|EOF|HMAC

Where:

2.1 TAG is the sequence of 4 ASCII characters "PWS3". This is to serve as a
quick way for the application to identify the database as a PasswordSafe
version 3 file. This tag has no cryptographic value.

2.2 SALT is a 256 bit random value, generated at file creation time.

2.3 P' is the "stretched key" generated from the user's passphrase and
the SALT, as defined by the hash-function-based key stretching
algorithm in [KEYSTRETCH] (Section 4.1), with SHA-256 [SHA256] as the
hash function, and ITER iterations (at least 2048, i.e., t = 11).

2.4 ITER is the number of iterations on the hash function to calculate P',
stored as a 32 bit little-endian value. This value is stored here in order
to future-proof the file format against increases in processing power.

2.5 H(P') is SHA-256(P'), and is used to verify that the user has the
correct passphrase.

2.6 B1 and B2 are two 128-bit blocks encrypted with Twofish [TWOFISH]
using P' as the key, in ECB mode. These blocks contain the 256 bit
random key K that is used to encrypt the actual records. (This has the
property that there is no known or guessable information on the
plaintext encrypted with the passphrase-derived key that allows an
attacker to mount an attack that bypasses the key stretching
algorithm.)

2.7 B3 and B4 are two 128-bit blocks encrypted with Twofish using P' as the
key, in ECB mode. These blocks contain the 256 bit random key L that is
used to calculate the HMAC (keyed-hash message authentication code) of the
encrypted data. See description of EOF field below for more details.
Implementation Note: K and L must NOT be related.

2.8 IV is the 128-bit random Initial Value for CBC mode.

2.9 All following records are encrypted using Twofish in CBC mode, with K
as the encryption key.

2.9.1 HDR: The database header. The header consists of one or more typed
fields (as defined in section 3.2), terminated by the 'END' type field. The
version number field is mandatory. Aside from the 'END' field, no
order is assumed on the field types.

2.9.2 R1..Rn: The actual database records. Each record consists of one or
more typed fields (as defined in Section 3.2), terminated by the 'END' type
field. The UUID, Title, and Password fields are mandatory. All non-
mandatory fields may either be absent or have zero length. When a field is
absent or zero-length, its default value shall be used. Aside from the
'END' field, no order is assumed on the field types.

2.10 EOF: The ASCII characters "PWS3-EOFPWS3-EOF" (note that this is
exactly one block long), unencrypted. This is an implementation convenience
to inform the application that the following bytes are to be processed
differently.

2.11 HMAC: The 256-bit keyed-hash MAC, as described in RFC2104, with SHA-
256 as the underlying hash function. The value is calculated over all of
the plaintext fields, that is, over all the data stored in all fields
(starting from the version number in the header, ending with the last field
of the last record). The key L, as stored in B3 and B4, is used as the hash
key value.

3. Fields: Data in PasswordSafe is stored in typed fields. Each field
consists of one or more blocks. The blocks are the blocks of the underlying
encryption algorithm - 16 bytes long for Twofish. The first block contains
the field length in the first 4 bytes (little-endian), followed by a one-
byte type identifier. The rest of the block contains up to 11 bytes of
record data. If the record has less than 11 bytes of data, the extra bytes
are filled with random values. The type of a field also defines the data
representation.

3.1 Data representations
3.1.1 UUID
 The UUID data type is 16 bytes long, as defined in RFC4122. Microsoft
 Windows has functions for this, and the RFC has a sample
 implementation.

3.1.2 Text
 Text is represented in UTF-8 encoding (as defined in RFC3629), with
 no byte order marker (BOM) and no end-of-string mark (e.g., null
 byte). Note that the latter isn't neccessary since the length of the
 field is provided explicitly. Note that ALL fields described as
 "text" are UTF-8 encoded unless explicitly stated otherwise.

3.1.3 Time
 Timestamps are stored as 32 bit, little endian, unsigned integers,
 representing the number of seconds since Midnight, January 1, 1970, GMT.
 (This is equivalent to the time_t type on Windows and POSIX. On the
 Macintosh, the value needs to be adjusted by the constant value 2082844800
 to account for the different epoch of its time_t type.)
 Note that future versions of this format may allow time to be
 specifed in 64 bits as well.

3.2 Field types for the PasswordSafe database header:
                                                 Currently
Name                        Value        Type    Implemented      Comments
--------------------------------------------------------------------------
Version                     0x00        2 bytes       Y              [1]
UUID                        0x01        UUID          Y              [2]
Non-default preferences     0x02        Text          Y              [3]
Tree Display Status         0x03        Text          Y              [4]
Timestamp of last save      0x04        time_t        Y              [5]
Who performed last save     0x05        Text          Y   [DEPRECATED 6]
What performed last save    0x06        Text          Y              [7]
Last saved by user          0x07        Text          Y              [8]
Last saved on host          0x08        Text          Y              [9]
Database Name               0x09        Text          Y              [10]
Database Description        0x0a        Text          Y              [11]
Database Filters            0x0b        Text          Y              [12]
Reserved                    0x0c        -                            [13]
Reserved                    0x0d        -                            [13]
Reserved                    0x0e        -                            [13]
Recently Used Entries       0x0f        Text                         [14]
Named Password Policies     0x10        Text                         [15]
Empty Groups                0x11        Text                         [16]
Reserved                    0x12        Text                         [13]
End of Entry                0xff        [empty]       Y              [17]

[1] The version number of the database format. For this version, the value
is 0x0310 (stored in little-endian format, that is, 0x10, 0x03).
PasswordSafe V3.01 introduced Format 0x0300
PasswordSafe V3.03 introduced Format 0x0301
PasswordSafe V3.09 introduced Format 0x0302
PasswordSafe V3.12 introduced Format 0x0303
PasswordSafe V3.13 introduced Format 0x0304
PasswordSafe V3.14 introduced Format 0x0305
PasswordSafe V3.19 introduced Format 0x0306
PasswordSafe V3.22 introduced Format 0x0307
PasswordSafe V3.25 introduced Format 0x0308
PasswordSafe V3.26 introduced Format 0x0309
PasswordSafe V3.28 introduced Format 0x030A
PasswordSafe V3.29 introduced Format 0x030B
PasswordSafe V3.29Y introduced Format 0x030C

[2] A universally unique identifier is needed in order to synchronize
databases, e.g., between a handheld pocketPC device and a
PC. Representation is as described in Section 3.1.1.

[3] Non-default preferences are encoded in a string as follows: The string
is of the form "X nn vv X nn vv..." Where X=[BIS] for binary, integer and
string respectively, nn is the numeric value of the enum, and vv is the
value, {1 or 0} for bool, unsigned integer for int, and a delimited string
for String. Only non-default values are stored. See PWSprefs.cpp for more
details.  Note: normally strings are delimited by the doublequote character.
However, if this character is in the string value, an arbitrary character
will be chosen to delimit the string. Version 0x0309 added database wide
special symbols to the Non-default preferences (0x02).

[4] If requested to be saved, this is a string of 1s and 0s indicating the
expanded state of the tree display when the database was saved. This can
be applied at database open time, if the user wishes, so that the tree is
displayed as it was. Alternatively, it can be ignored and the tree
displayed completely expanded or collapsed. Note that the mapping of
the string to the display state is implementation-specific. Introduced
in format 0x0301.

[5] Representation is as described in Section 3.1.3. Note that prior
to PasswordSafe 3.09, this field was mistakenly represented as an
eight-byte hexadecimal ASCII string. Implementations SHOULD attempt to
parse 8-byte long timestamps as a hexadecimal ASCII string
representation of the timestamp value.

[6] Text saved in the format: nnnnu..uh..h, where: 
    nnnn = 4 hexadecimal digits giving length of following user name field
    u..u = user name
    h..h = host computer name
    Note: As of format 0x0302, this field is deprecated, and should be
    replaced by fields 0x07 and 0x08. In databases prior to format
    0x0302, this field should be maintained. 0x0302 and later may
    either maintain this field in addition to fields 0x07 and 0x08,
    for backwards compatability, or not write this field. If both this
    field and 0x07, 0x08 exist, they MUST represent the same values.

[7] Free form text giving the application that saved the database.
For example, the Windows PasswordSafe application will use the text
"Password Safe Vnn.mm", where nn and mm are the major and minor
version numbers. The major version will contain only the significant
digits whereas the minor version will be padded to the left with
zeroes e.g. "Password Safe V3.02".

[8] Text containing the username (e.g., login, userid, etc.) of the
user who last saved the database, as determined by the appropriate
operating-system dependent function. This field was introduced in
format version 0x0302, as a replacement for field 0x05. See Comment
[6].

[9] Text containing the hostname (e.g., machine name, hostid, etc.) of the
machine on which the database was last saved, as determined by the
appropriate operating-system dependent function. This field was
introduced in format version 0x0302, as a replacement for field
0x05. See Comment [6].

[10] Database name. A logical name for a database which can be used by
applications in place of the possibly lengthy filepath notion. Note
that this field SHOULD be limited to what can be displayed in a single
line. This field was introduced in format version 0x0302.

[11] Database Description. A purely informative description concerning
the purpose or other practical use of the database. This field was
introduced in format version 0x0302.

[12] Specfic filters for this database.  This is the text equivalent to
the XML export of the filters as defined by the filter schema. The text 
'image' has no 'print formatting' e.g. tabs and carraige return/line feeds,
since XML processing does not require this. This field was introduced in 
format version 0x0305.

[13] Values marked 'Reserved' are know to have been used by customized
versions of PasswordSafe. To ensure compatability between versions,
use of these values should be avoided.

[14] A list of the UUIDs (32 hex character representation of the 16 byte field)
of the recently used entries, prefixed by a 2 hex character representation
of the number of these entries (right justified and left filled with zeroes).
The size of the number of entries field gives a maximum number of entries of 255,
however the GUI may impose further restrictions e.g. Windows MFC UI limits this
to 25. The first entry is the most recent entry accessed. This field was
introduced in format version 0x0307.

[15] This field allows multiple Password Policies per database.  The format is:
    "NN{LLxxx...xxxffffnnnllluuudddsssMMSSS...SSS}"
where:
     NN = 2 hexadecimal digits giving number of password policies in
    field (max. 255).
     
   Each entry is of the form:
     LL = 2 hexadecimal digits giving length of the policy name in bytes
     xxx...xxx = The policy name (maximum 255 bytes)
     ffff = 4 hexadecimal digits representing the following flags
        UseLowercase        0x8000
        UseUppercase        0x4000
        UseDigits           0x2000
        UseSymbols          0x1000
        UseHexDigits        0x0800 (if set, then no other flags can be set)
        UseEasyVision       0x0400
        MakePronounceable   0x0200
        Unused              0x01ff
    nnn  = 3 hexadecimal digits password total length
    lll  = 3 hexadecimal digits password minimum number of lowercase characters
    uuu  = 3 hexadecimal digits password minimum number of uppercase characters
    ddd  = 3 hexadecimal digits password minimum number of digit characters
    sss  = 3 hexadecimal digits password minimum number of symbol characters
    MM = 2 hexadecimal digits giving the length in bytes of allowed special
         symbols, or zero to specify the database default special symbol
         set.
    SSS...SSS = List of allowed symbols in this policy (maximum 255 bytes)
    
This field was introduced in format version 0x030A.

[16] This fields contains the name of an empty group that cannot be constructed
from entries within the database. Unlike other header fields, this field can appear
multiple times.

This field was introduced in format version 0x030B.

[17] An explicit end of entry field is useful for supporting new fields
without breaking backwards compatability.

3.3 Field types for database Records:
                                                 Currently
Name                        Value        Type    Implemented      Comments
--------------------------------------------------------------------------
UUID                        0x01        UUID          Y              [1]
Group                       0x02        Text          Y              [2]
Title                       0x03        Text          Y
Username                    0x04        Text          Y
Notes                       0x05        Text          Y
Password                    0x06        Text          Y              [3,4]
Creation Time               0x07        time_t        Y              [5]
Password Modification Time  0x08        time_t        Y              [5]
Last Access Time            0x09        time_t        Y              [5,6]
Password Expiry Time        0x0a        time_t        Y              [5,7]
*RESERVED*                  0x0b        4 bytes       -              [8]
Last Modification Time      0x0c        time_t        Y              [5,9]
URL                         0x0d        Text          Y              [10]
Autotype                    0x0e        Text          Y              [11]
Password History            0x0f        Text          Y              [12]
Password Policy             0x10        Text          Y              [13]
Password Expiry Interval    0x11        2 bytes       Y              [14]
Run Command                 0x12        Text          Y
Double-Click Action         0x13        2 bytes       Y              [15]
EMail address               0x14        Text          Y              [16]
Protected Entry             0x15        1 byte        Y              [17]
Own symbols for password    0x16        Text          Y              [18]
Shift Double-Click Action   0x17        2 bytes       Y              [15]
Password Policy Name        0x18        Text          Y              [19]
End of Entry                0xff        [empty]       Y              [20]

[1] Per-record UUID to assist in sync, merge, etc. Representation is
as described in Section 3.1.1.

[2] The "Group" supports displaying the entries in a tree-like manner.
Groups can be hierarchical, with elements separated by a period, supporting
groups such as "Finance.credit cards.Visa", "Finance.credit
cards.Mastercard", Finance.bank.web access", etc. Dots entered by the user
should be "escaped" by the application.

[3] If the entry is an alias, the password will be saved in a special form 
of "[[uuidstr]]", where "uuidstr" is a 32-character representation of the 
alias' associated base entry's UUID (field type 0x01).  This representation
is the same as the standard 36-character string representation as defined in 
RFC4122 but with the four hyphens removed. If an entry with this UUID is not
in the database, this is treated just as an 'unusual' password.  The alias
will only use its base's password entry when copying it to the clipboard or
during Autotype.

[4] If the entry is a shortcut, the password will be saved in a special form 
of "[~uuidstr~]", where "uuidstr" is a 32-character representation of the 
shortcut's associated base entry's UUID (field type 0x01).  This representation
is the same as the standard 36-character string representation as defined in 
RFC4122 but with the four hyphens removed. If an entry with this UUID is not
in the database, this is treated just as an 'unusual' password. The shortcut
will use all its base's data when used in any action.  It has no fields of
its own.

[5] Representation is as described in Section 3.1.3.

[6] This will be updated whenever any part of this entry is accessed
i.e., to copy its username, password or notes to the clipboard; to
perform autotype or to browse to url.

[7] This will allow the user to enter an expiry date for an entry. The
application can then prompt the user about passwords that need to be
changed. A value of zero means "forever".

[8] Although earmarked for Password Policy, the coding in versions prior
to V3.12 does not correctly handle the presence of this field.  For this
reason, this value cannot be used for any future V3 field without causing
a potential issue when a user opens a V3.12 or later database with program
version V3.11 or earlier.  See note [14].

[9] This is the time that any field of the record was modified, useful for
merging databases.

[10] The URL will be passed to the shell when the user chooses the "Browse
to" action for this entry. In version 2 of the format, this was extracted
from the Notes field. By placing it in a separate field, we are no longer
restricted to a URL - any action that may be executed by the shell may be
specified here.

[11] The text to be 'typed' by PasswordSafe upon the "Perform Autotype"
action maybe specified here. If unspecified, the default value of
'username, tab, password, tab, enter' is used. In version 2 of the format,
this was extracted from the Notes field. Several codes are recognized here,
e.g, '%p' is replaced by the record's password. See the user documentation
for the complete list of codes. The replacement is done by the application
at runtime, and is not stored in the database.

[12] Password History is an optional record. If it exists, it stores the
creation times and values of the last few passwords used in the current
entry, in the following format:
    "fmmnnTLPTLP...TLP"
where:
    f  = {0,1} if password history is on/off
    mm = 2 hexadecimal digits max size of history list (i.e. max = 255)
    nn = 2 hexadecimal digits current size of history list
    T  = Time password was set (time_t written out in %08x)
    L  = 4 hexadecimal digit password length (in TCHAR)
    P  = Password
No history being kept for a record can be represented either by the lack of
the PWH field (preferred), or by a header of _T("00000"):
    flag = 0, max = 00, num = 00
Note that 0aabb, where bb <= aa, is possible if password history was enabled
in the past and has then been disabled but the history hasn't been cleared.

[13] This field allows a specific Password Policy per entry.  This field
is mutually exclusive with the policy name field [0x18]. The format is:
    "ffffnnnllluuudddsss"
where:
     ffff = 4 hexadecimal digits representing the following flags
        UseLowercase        0x8000 
        UseUppercase        0x4000 
        UseDigits           0x2000 
        UseSymbols          0x1000 
        UseHexDigits        0x0800 (if set, then no other flags can be set)
        UseEasyVision       0x0400
        MakePronounceable   0x0200
        Unused              0x01ff
    nnn  = 3 hexadecimal digits password total length
    lll  = 3 hexadecimal digits password minimum number of lowercase characters
    uuu  = 3 hexadecimal digits password minimum number of uppercase characters
    ddd  = 3 hexadecimal digits password minimum number of digit characters
    sss  = 3 hexadecimal digits password minimum number of symbol characters

[14] Password Expiry Interval, in days, before this password expires. Once set,
this value is used when the password is first generated and thereafter whenever
the password is changed, until this value is unset.  Valid values are 1-3650
corresponding to up to approximately 10 years.  A value of zero is equivalent to
this field not being set.

[15] A two byte field contain the value of the Double-Click Action and Shift
Double-Click Action'preference value' (0xff means use the current
Application default):
Current 'preference values' are:
    CopyPassword           0
    ViewEdit               1
    AutoType               2
    Browse                 3
    CopyNotes              4
    CopyUsername           5
    CopyPasswordMinimize   6
    BrowsePlus             7
    Run Command            8
    Send email             9

[16] Separate Email address field as per RFC 2368 (without the 'mailto:'
prefix. This field was introduced in version 0x0306 (PasswordSafe V3.19).

[17] Entry is protected, i.e., the entry cannot be changed or deleted
while this field is set. This field was introduced in version 0x0308
(PasswordSafe V3.25).  This a single byte. An absent field or a zero
valued field means that the entry is not protected. Any non-zero value
means that the entry is protected.

[18] Each entry can now specify its own set of allowed special symbols for
password generation.  This overrides the default set and any database specific
set. This field is mutually exclusive with the policy name field
[0x18].  This was introduced in version 0x0309 (PasswordSafe V3.26).

[19] Each entry can now specify the name of a Password Policy saved in
the database header for password generation. This field is mutually
exclusive with the specific policy field [0x10] and with the Own
symbols for password field [0x16]. This was introduced in version
0x030A (PasswordSafe V3.28).

[20] An explicit end of entry field is useful for supporting new fields
without breaking backwards compatability.

4. Extensibility

4.1 Forward compatability: Implementations of this format SHOULD NOT
discard or report an error when encountering a field of an unknown
type. Rather, the field(s) type and data should be read, and preserved
when the database is saved.

4.2 Field type identifiers: This document specifies the field type
identifiers for the current version of the format. Compliant
implementations MUST support the mandatory fields, and SHOULD support
the other fields described herein. Future versions of the format may
specify other type identifiers.

4.2.1 Application-unique type identifiers: The type identifiers
0xc0-0xdf are available for application developers on a first-come
first-serve basis. Application developers interested in reserving a
type identifier for their application should contact the maintainer of
this document (Currently the PasswordSafe project administrator at
SourceForge).

4.2.2 Application-specific type identifiers: The type identifiers
0xe0-0xfe are reserved for implementation-specific purposes, and will
NOT be specified in this or future versions of the format
description.

4.2.3 All unassigned identifiers except as listed in the previous two
subsections are reserved, and should not be used by other
implementations of this format specification in the interest of
interoperablity.

5. References:
[TWOFISH] http://www.schneier.com/paper-twofish-paper.html
[SHA256]
http://csrc.nist.gov/publications/fips/fips180-2/fips180-2withchangenotice.pdf
[KEYSTRETCH] http://www.schneier.com/paper-low-entropy.pdf

End of Format description.

```



