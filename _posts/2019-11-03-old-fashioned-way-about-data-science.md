---
layout: post
title:  "Old-fashioned Way About Data Science"
tags: linux cli shell gnuplot
author: "freefd"
---

The usual Pythonic way to get a data series is sometimes not the fastest. In some cases, it's enough to have only 3-6 Linux CLI tools to get a necessary data from Web and parse it in a proper way. Let's look at such an approach on how to count the number of released RFC's per year and draw the graph in a terminal.

## All you need is text
The important question at the start is always "Where can I get the data source?". Fortunately, in 2019<sup>th</sup> you can find the organized list for [all of RFCs](https://www.rfc-editor.org/) <sup id="a1">[1](#f1)</sup>.

At the first glance it seems that we need to parse HTML or XML data, but it's not. Actually, we need to parse a simple text/plain data which should be in the same format for all entries. Here comes our first helper - [w3m](http://w3m.sourceforge.net/) <sup id="a2">[2](#f2)</sup> CLI text-based browser.
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | head -50
RFC Index

...
avoided for brevity
...

See the RFC Editor Web page for more information.

RFC Index

Num  Information
0001 Host Software S. Crocker [ April 1969 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0001)
0002 Host software B. Duvall [ April 1969 ] (TXT, PDF, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0002)
0003 Documentation conventions S.D. Crocker [ April 1969 ] (TXT, HTML) (Obsoleted-By RFC0010) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0003)
```
That's how we find strictly-formed entries as follows:

> `NUMBER DESCRIPTION [ DATE ] OTHER_TECHNICAL_INFO`

and we can extract the date information using [awk](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) <sup id="a3">[3](#f3)</sup> - a DSL designed for text processing. I prepared a test file from a small fragment of the collected data:
```bash
~> cat test-entries
0506 FTP command naming problem M.A. Padlipsky [ June 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0506)
0507 Not Issued
0508 Real-time data transmission on the ARPANET L. Pfeifer, J. McAfee [ May 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0508)
0509 Traffic statistics (April 1973) A.M. McKenzie [ April 1973 ] (TXT, HTML) (Status: UNKNOWN) (Stream: Legacy) (DOI: 10.17487/RFC0509)

~> awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries
 June 1973 
 May 1973 
 April 1973 
```
Please note that `^[0-9][0-9][0-9][0-9]` is used instead of `^[0-9]{4}` because [mawk](https://invisible-island.net/mawk/) <sup id="a4">[4](#f4)</sup> [doesn't support](https://github.com/ThomasDickey/original-mawk/issues/25) <sup id="a5">[5](#f5)</sup> repetition.

Since now we able to extract human-readable dates, let's convert them to numeric format with [date](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html) <sup id="a6">[6](#f6)</sup> utility during implicit loop given from [xargs](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/xargs.html) <sup id="a7">[7](#f7)</sup>:
```bash
~> # Years extraction
~> awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%Y"
1973
1973
1973
~> # Months extraction
~> awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m"
06
05
04
```
The next simple step is sorting ([sort](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/sort.html) <sup id="a8">[8](#f8)</sup> tool) and counting unique ([uniq](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/uniq.html) <sup id="a9">[9](#f9)</sup> tool) values:
```bash
~> awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%Y" | sort -n | uniq -c
      3 1973

~> awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' test-entries | xargs -I {} env TZ=Europe/London date -d'01 {}' +"%m" | sort -n | uniq -c
      1 04
      1 05
      1 06
```
We're ready to put it all together over pipes:
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%Y" | sort -n | uniq -c
      1 1968
     25 1969
     58 1970
    182 1971
    134 1972
    162 1973
     60 1974
     24 1975
     11 1976
     20 1977
      8 1978
      7 1979
     17 1980
     29 1981
     37 1982
     49 1983
     39 1984
     41 1985
...
```

## Drawing in terminal
Bare numbers aren't very comfortable for analysis and thus we'll use [gnuplot](http://www.gnuplot.info/) <sup id="a10">[10](#f10)</sup> utility to draw graphs in the following [configuration](http://www.bersch.net/gnuplot-doc/gnuplot.html) <sup id="a11">[11](#f11)</sup>:

```bash
gnuplot -e "set term dumb size 145, 25; set xtics 3; plot '-' with lines notitle"
```
It'll read the STDIN stream and draw a 145x25 graph right in the terminal. Putting it all together one more time:

```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%Y" | sort -n | uniq -c | gnuplot -e "set term dumb size 145, 25; set xtics 3; plot '-' with lines notitle"
line 52: warning: Too many axis ticks requested (>2e+02)

                                                                                                                                                 
  2020 +-------------------------------------------------------------------------------------------------------------------------------------+   
       |                                                   ****************************************                                          |   
       |                                                                               ****************                                      |   
  2010 |-+                                                                                       *************************                 +-|   
       |                                                                                  ************                                       |   
       |                                                                                        *********************************************|   
       |                                                           *****************************                                             |   
  2000 |-+                                                     **************************                                                  +-|   
       |                                           ****************************                                                              |   
       |                                     *****************                                                                               |   
  1990 |-+             ************************                                                                                            +-|   
       |            ***                                                                                                                      |   
       |      ******                                                                                                                         |   
       |         *****                                                                                                                       |   
  1980 |-********                                                                                                                          +-|   
       | *****                                                                                                                               |   
       |     ******************************************                                                                                      |   
  1970 |-+         ******************************************                                                                              +-|   
       |***********                                                                                                                          |   
       |                                                                                                                                     |   
       |                                                                                                                                     |   
  1960 +-------------------------------------------------------------------------------------------------------------------------------------+
```
As you can see, something is going wrong. This is because gnuplot expects the first column as the X-axis and the second as the Y-axis. We need to swap our columns with each other:
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%Y" | sort -n | uniq -c | awk '{print $2" "$1}'
1968 1
1969 25
1970 58
1971 182
1972 134
1973 162
1974 60
1975 24
1976 11
1977 20
1978 8
1979 7
1980 17
1981 29
1982 37
1983 49
1984 39
1985 41
...
```
And rerun our graph plotting:
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%Y" | sort -n | uniq -c | awk '{print $2" "$1}' | gnuplot -e "set term dumb size 145, 25; set xtics 3; plot '-' with lines notitle"

                                                                                                                                                 
  500 +--------------------------------------------------------------------------------------------------------------------------------------+   
      |       +       +       +       +       +       +       +       +      +       +       +       +       +       +       +       +       |   
  450 |-+                                                                                                  *                               +-|   
      |                                                                                                   **                                 |   
  400 |-+                                                                                                 * *                              +-|   
      |                                                                                                  *  *          **                    |   
  350 |-+                                                                                                *  *        **  *                 +-|   
      |                                                                                                 *    *      *     ***   **           |   
  300 |-+                                                                                             **     ***    *         **  ******   +-|   
      |                                                                                   **         *          ****         *               |   
  250 |-+                                                                              ***  *       *                                   *  +-|   
      |                                                                              **     *     **                                     *   |   
      |                                                                             *        * ***                                        *  |   
  200 |-+     ***                                                         **      **         **                                            *-|   
      |       *   **                                                    **  *  ***                                                          *|   
  150 |-+    *   *  *                                                  *     **                                                            +-|   
      |      *       *                                                 *                                                                     |   
  100 |-+   *        *                                             ****                                                                    +-|   
      |     *         ***                                        **                                                                          |   
   50 |-+ **                              ************ **********                                                                          +-|   
      | **    +       +  **  ***     *****    +       *       +       +      +       +       +       +       +       +       +       +       |   
    0 +--------------------------------------------------------------------------------------------------------------------------------------+   
     1968    1971    1974    1977    1980    1983    1986    1989    1992   1995    1998    2001    2004    2007    2010    2013    2016    2019
```
Now it looks nice. The same for months where we see the expected deviations for July and November/December, the most productivity release dates are in March/April:
```bash
~> w3m -cols 1024 -dump https://www.rfc-editor.org/rfc-index.html | awk -F'[\[|\]]' '/^[0-9][0-9][0-9][0-9]/ && !/Not Issued/{print $2}' | xargs -I {} env TZ=Europe/London date -d'01{}' +"%m" | sort -n | uniq -c | awk '{print $2" "$1}' | gnuplot -e "set term dumb size 145, 25; set xtics 1; plot '-' with lines notitle smooth csplines"

                                                                                                                                                 
  850 +--------------------------------------------------------------------------------------------------------------------------------------+   
      |           +            +           +           +           +            +           +           +           +            +           |   
      |                                                                                                                                      |   
  800 |-+                     *********                                                                                                    +-|   
      |                    ***         **                                                                                                    |   
      |                   *              ***                                                                                                 |   
      |                  *                  ***               *****                         ****                   *                         |   
  750 |-+              **                      ***       *****     **                     **    ***             *** ***                    +-|   
      |               *                           *******            *                   *         *           *       **                    |   
      |              *                                                *                 *           *        **          *                   |   
  700 |****       ***                                                  *               *             **    **             *                +-|   
      |    *******                                                      *             *                ****               *                  |   
      |                                                                  *           *                                     *                 |   
      |                                                                   **         *                                      *                |   
  650 |-+                                                                   *      **                                        *             +-|   
      |                                                                      *    *                                           *              |   
      |                                                                      *****                                            *              |   
  600 |-+                                                                                                                      **          +-|   
      |                                                                                                                          *           |   
      |                                                                                                                           *          |   
      |           +            +           +           +           +            +           +           +           +            + **        |   
  550 +--------------------------------------------------------------------------------------------------------------------------------------+   
      1           2            3           4           5           6            7           8           9           10           11          12
```

## References
<b id="f1">1</b>. [RFC Editor](https://www.rfc-editor.org/) [↩](#a1)<br/>
<b id="f2">2</b>. [Text-based web browser w3m](http://w3m.sourceforge.net/) [↩](#a2)<br/>
<b id="f3">3</b>. ["Aho, Weinberger and Kernighan" domain-specific language](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) [↩](#a3)<br/>
<b id="f4">4</b>. [awk originally written by Mike Brennan](https://invisible-island.net/mawk/) [↩](#a4)<br/>
<b id="f5">5</b>. [Built-in regex's do not support brace-expressions](https://github.com/ThomasDickey/original-mawk/issues/25) [↩](#a5)<br/>
<b id="f6">6</b>. [date - write the date and time](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html) [↩](#a6)<br/>
<b id="f7">7</b>. [xargs - construct argument lists and invoke utility](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/xargs.html) [↩](#a7)<br/>
<b id="f8">8</b>. [sort - sort, merge, or sequence check text files](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/sort.html) [↩](#a8)<br/>
<b id="f9">9</b>. [uniq - report or filter out repeated lines](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/uniq.html) [↩](#a9)<br/>
<b id="f10">10</b>. [gnuplot - portable command-line driven graphing utility](http://www.gnuplot.info/) [↩](#a10)<br/>
<b id="f11">11</b>. [gnuplot documentation](http://www.bersch.net/gnuplot-doc/gnuplot.html) [↩](#a11)<br/>
