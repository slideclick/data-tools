[introduction](#intro) | [encodings](#encodings) | [newlines](#newlines) | [relational formats](#relational-fmt) | [hierarchical formats](#hierarchical-fmt)

<a name="intro"/>
# Introduction

The tools in this repo come with man pages which can be installed or browsed on [GitHub](https://github.com/clarkgrubb/data-tools/tree/master/doc).

The traditional command line tools `awk`, `sort`, and `join` can be used to implement a poor man's relational algebra.  This turns out to be useful, because our tabular data isn't always in a SQL database.  Sometimes it's in a flat file.  Anyway, dealing with this kind of data is the central theme of the tools in this repository.

<a name="encodings"/>
# Encodings

All tools expect and produce UTF-8 (or 8-bit clean ASCII) encoded data.  Use
`iconv` if you need to deal with a different encoding, e.g:

    iconv -t UTF-8 -f UTF-16 /etc/passwd > /tmp/pw.utf16
    
To get a list of supported encodings:
    
    iconv -l

Not all sequences of bytes are valid UTF-8, and the tools with throw exceptions when invalid bytes are encountered.  A drastic way to deal with the problem is to strip the invalid bytes:

    iconv -c -f UTF-8 -t UTF-8 < INPUT_FILE > OUTPUT_FILE

How to find non-ASCII bytes:

    grep --color='auto' -P -n "[\x80-\xFF]+"

The `-P` option is not available in the version of `grep` distributed with Mac OS X, however.  One can use the `highlight` command in this repo:

    highlight '[\x80-\xFF]+'

How to find invalid UTF-8 bytes?

When a file is in an unknown encoding, one is compelled to inspect it byte-by-byte.
Three tools are suggested.

    od -b
    od -c
    cat -te
    
`od -b` will display the input in octal bytes.  If the input is thought to be encoded in one of the many 8-bit
extensions of ASCII, `od -c` is useful, since it renders printable ASCII and uses octal bytes for everything else.
`cat -t` renders printable ASCII and newlines, and uses `^` notation for other control characters.  Some versions of `cat -t`
use Emacs style `M-X` notation for upper 8-bit bytes.  In this case, `X` will be what `cat -t` would have used to render
the character if the upper bit were zero, with the exception of `^J` being used for newline.

A binary editor can be a useful thing to have.  The repo will build a version of [hexedit](http://rigaux.org/hexedit.html) to which a [patch](http://www.volkerschatz.com/unix/hexeditpatch.html) supporting aligned search has been applied.

Although utf-8 is great, it is also difficult.  It contains characters with strange properties (combing characters, right-to-left characters)

    utf8-viewer
   
If you want to see the character for a Unicode point, the following works in `zsh`:

     $ echo $'\u03bb'
     
If you are using a different shell but have access to `python` or `ruby`:
     
     $ python -c 'print(u"\u03bb")'
     
     $ ruby -e 'puts "\u03bb"'
 
How to lookup a Unicode point:

    $ curl ftp://ftp.unicode.org/Public/UNIDATA/UnicodeData.txt > /tmp/UnicodeData.txt
    
    $ awk -F';' '$1 == "03BB"' /tmp/UnicodeData.txt 
    03BB;GREEK SMALL LETTER LAMDA;Ll;0;L;;;;;N;GREEK SMALL LETTER LAMBDA;;039B;;039B

`UnicodeData.txt` is a useful file, and possibly it deserves a dedicated path on your file system.  

The first three fields are "Point", "Name", and "[General Category](http://www.unicode.org/reports/tr44/#General_Category_Values)".  

<a name="newlines"/>
# Newlines

[eol markers](#eol-markers) | [set operations](#set-op) | [highlighting](#highlighting) | [sequences](#seq) | [sampling](#sampling)

<a name="eol-markers"/>
## EOL Markers

The tools use LF as the end-of-line marker in output.  The tools will generally
handle other EOL markers in input correctly.  See [this](http://www.unicode.org/standard/reports/tr13/tr13-5.html)
for a list of the Unicode characters that should be treated as EOL markers.
Use `unix2dos` (see if your package manager has the package `dos2unix`) to convert the output of one of the tools to CRLF style EOL markers.

    tr '\n' '\r'
    tr -d '\r'
    
    dos2unix
    unix2dos
   
Tools are provided for finding the lines which two files share in common, or which are unique to the first file:

<a name="set-op"/>
## Set Operations

    set-intersect FILE1 FILE2
    set-diff FILE1 FILE2
    
The `cat` command can be used to find the union of two files, with an optional `sort -u` to remove duplicate lines:
    
    cat FILE1 FILE2 | sort -u

<a name="highlighting"/>
## Highlighting

When inspecting files at the command line, `grep` and `less` are invaluable.  Their man pages reward careful study.
An interesting feature of `grep` is the ability to hightlight the search pattern in red:

    grep --color=always root /etc/passwd
    
The `highlight` command does the same thing, except that it also prints lines which don't match
the pattern.  Also it supports multiple patterns, each with its own color:
    
    highlight --red root --green daemon --blue /bin/bash /etc/passwd

Both `grep` and `highlight` use [ANSI Escapes](http://www.ecma-international.org/publications/standards/Ecma-048.htm).  If you are paging through the output, use `less -R` to render the escape sequences correctly.

<a name="seq"/>
## Sequences

The `seq` command can generate a newline delimited arithmetic sequence:

    $ seq 1 3
    1
    2
    3

Zero-padded:

    $ seq -w 08 11
    08
    09
    10
    11
    
Step values other than one:

    $ seq 1 .5 2
    1
    1.5
    2

The `seq` is useful in conjunction with the a shell for loop.  This will create a hundred empty files:

    $ for i in $(seq -w 1 100); do touch foo.$i; done

It is also useful to at times to be able to iterate through a sequence of dates.

    date-seq

<a name="sampling"/>
## Sampling

    sort -R | head -10
    awk 'rand() < 0.01'
    sample

<a name="relational-fmt"/>
# Relational Formats

[tsv](#tsv) | [csv](#csv) | [json](#relational-json) | [xlsx](#xlsx)

As mentioned previously, much that can be done with a SQL SELECT statement in a database can also be done with `awk`, `sort`, and `join`.  If you are not familiar with the commands, consider reading the man pages.

Relational data can be stored in flat files in a variety of ways.  On Unix, the `/etc/passwd` file stores records one per line, with colons (:) separating the seven fields.  We can use `awk` to query the file.

Get the root entry from `/etc/passwd`:

    awk -F: '$1 == "root"' /etc/passwd

Count the number of users by their login shell:

    awk -F: '{cnt[$7] += 1} END {for (sh in cnt) print sh, cnt[sh]}' /etc/passwd

The `/etc/passwd` file format, though venerable, has a bit of an adhoc flavor.  We discuss four widely used formats

<a name="tsv"/>
## TSV

The IANA, which registered MIME types, has a [specification for TSV](http://www.iana.org/assignments/media-types/text/tab-separated-values).  Records are newline delimited and fields are tab-delimited.  There is no mechanism for escaping or quoting tabs and newlines.  Despite this limitation, we prefer to convert the other formats to TSV because `awk`, `sort`, and `join` cannot easily manipulate the other formats.

Tabs receive criticism, and deservedly, because they are indistinguishable as normally rendered from spaces, which can cause cryptic errors.   Trailing spaces in fields can be hidden by tabs, causing joins to mysteriously fail, for example.  `cat -t` can used to expose trailing spaces.  Note that trailing spaces at the end of a line can also be hidden.  `cat -e` can be used to find these. The `strip-columns` tool can be used to clean up a TSV file.

The fact that tabs are visually identical to spaces, means that in many applications they can be replaced by spaces, which is why tabs are more likely to be available for delimiting fields than any other printing character.  One could ofcourse use a non-printing character, but most applications do not display non-printing characters well.  Here is how to align the columns of a tab delimited file:

    tr ':' '\t' < /etc/passwd | column -t -s $'\t'

The default field separator for `awk` is whitespace.  This is results in an error-prone situation, because the default behavior sometimes works on TSV files.  The correct way to use `awk` on a TSV is like this:

    awk 'BEGIN {FS="\t"; OFS="\t"} ...'

Because this is a bit tedious, the repo contains a `tawk` command which uses tabs by default:

    tawk '...'

The IANA spec says that a TSV file must have a header.  This is a good practice, since it makes the data self-describing.  Unfortunately the header is at times inconvenient; when sorting the file, for example.  The repo provides the `header-sort` command to sort a file while keeping the header in place.  Also, when we must remove the header, we label the file with a `.tab` suffix instead of a `.tsv` suffix, though this is not a universal practice.

Even if a file has a header, `awk` scripts must refer to columns by number instead of name.  The following code displays the header names with their numbers:

    head -1 foo.tsv | tr '\t' '\n' | nl

*generating and parsing TSV with split and join. looping over the fields and outputing a space after each field is a bad practice since it results in an invsible trailing space on the last fields*

<a name="csv"/>
## CSV

    csv-to-tsv
    tsv-to-cvs

The CSV spec.

CSV files do not necessarily have headers.  This is perhaps because CSV files are an export format for spreadsheets.

*TSV with quotes*

* csvkit

<a name="relational-json"/>
## JSON

JSON ([json.org](http://json.org/)) is discussed more in the hierarchical section.  MongoDB has popularized its use for relational (or near relational) data.  The MongoDB export format is a file of serialized JSON objects, one per line.  Whitespace can be added or removed anywhere to a serialized JSON object without changing the data the JSON object represents (except inside strings, and newlines must be escaped in strings).  This is why each JSON object can be written on a single line.

The following tools are provided to convert CSV or TSV files to the MongoDB export format.  In the case of `csv-to-json`, the CSV file must have a header:

    csv-to-json
    tsv-to-json

<a name="xlsx"/>
## XLSX

XLSX is the default format used by Excel since 2007.  Most other spreadsheet applications can read it.  It is a standardized format, and it probably the most commonly encountered spreadsheet format.

XLSX is a ZIP archive of mostly XML files.  The `unzip -l` command can be used to inspect the archive.

    xlsx-to-csv

<a name="hierarchical-fmt"/>
# Hierarchical Formats

## JSON

    json-awk
    python -mjson.tool


## XML and HTML
 
    dom-awk
    xmllint
