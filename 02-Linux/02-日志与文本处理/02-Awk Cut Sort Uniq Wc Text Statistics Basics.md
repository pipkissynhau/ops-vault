# 02-awk, cut, sort, uniq, wc Text Statistics Basics

#Linux #LogAnalysis #TextProcessing #awk #cut #sort #uniq #wc #FieldExtract #StatisticalAnalysis #Transport

---

## Recommended Path

01-Linux Basics and Host Maintenance/02-Log and Text Processing/02-awk, cut, sort, uniq, wc Text Statistics Basics.md

---

## One, Document Explanation

This document organizes commonly used commands in Linux log analysis and text processing: **awk, cut, sort, uniq, wc**.

This article focuses on:

- awk Column Extraction
- awk Extracting First Column, Multiple Columns, Last Column, Penultimate Column
- awk Specifying Separator
- awk Conditional Filtering
- awk Multi-Condition Judgment
- awk Regular Expression Matching
- awk Counting, Summing, Averaging
- awk Passing Variables
- cut Simple Column Slicing
- sort Sorting
- uniq Deduplication and Repetition Statistics
- wc Line, Word, Byte Count Statistics
- sort + uniq + wc Common Combinations
- Text Statistics TopN General Pattern

This article is the second in the Log and Text Processing series, mainly solving:

```text
How to extract fields from a log or text, filter conditions, number of statistics, sorting Heavy
```

The goal is:

Be able to understand text field structure

→ Be able to extract specified columns with awk

→ Be able to filter logs with awk based on conditions

→ Be able to perform simple statistics with awk

→ Be able to perform simple field slicing with cut

→ Be able to perform sorting and deduplication statistics with sort and uniq

→ Be able to perform basic counting with wc

→ Be able to combine commands to complete common log statistics analysis

---

## Two, Core Command Location

The commands involved in this article are located as follows:

```text
awk
→ By column extraction, condition judgement, statistics

cut
→ Simple tangent column for fixed separator text

sort
→ Sort

uniq
→ Heavy, double counting.

wc
→ Count
```

One-sentence understanding:

```text
awk More suitable for complex fields

cut More suitable for simple rows

sort Sorting

uniq I'm responsible for weighting and statistical duplication.

wc Responsible for counting lines, words, bytes
```

---

## Three, Basic Concepts

---

## 1. Columns in awk

awk defaults to splitting fields by whitespace characters.

Example text:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123
```

In awk:

```text
$1
→ First column

$2
→ Column 2

$3
→ Column 3

$0
→ Line

$NF
→ Last row

$(NF-1)
→ 2nd Last Column
```

Example:

```bash
awk '{print $1}' access.log
```

Meaning:

```text
Print access.log First row
```

---

## 2. NF and NR in awk

### NF

`NF` Represents the total number of columns in the current line.

Example:

```bash
awk '{print NF}' file
```

Meaning:

```text
Print the number of fields per line
```

### NR

`NR` Represents the current line number being processed.

Example:

```bash
awk '{print NR,$0}' file
```

Meaning:

```text
Print line numbers and line contents
```

---

## 3. Relationship between sort and uniq

`uniq` Can only process adjacent duplicate lines.

Therefore, it's generally necessary to first `sort`, then `uniq`.

Common writing:

```bash
sort file | uniq -c
```

Meaning:

```text
Sort first and line up the same content.

Again. uniq -c Number of statistical repetitions
```

---

## 4. General Pattern for Text Statistics

The most common statistics pattern is:

```bash
awk '{print Column}' file | sort | uniq -c | sort -nr | head
```

Meaning:

```text
awk
→ Extract Fields

sort
→ Sort, so that the same content is adjacent

uniq -c
→ Number of statistical repetitions

sort -nr
→ Sort in reverse order of numbers

head
→ Take the first few.
```

Suitable scenarios:

```text
Most visited by statistics IP

Most visited by statistics URL

Statistical status code distribution

Most errors in statistics

Statistics for a column TopN
```

---

## Four, awk: Column Extraction

---

## Scenario 1: Extract First Column

### Command

```bash
awk '{print $1}' file
```

### Explanation

Commonly used for extracting:

```text
IP Address

Username

First field

Client address in log

First column in command output
```

### Example

Extracting the first column (IP) from Nginx access.log:

```bash
awk '{print $1}' access.log
```

---

## Scenario 2: Extract Second Column

### Command

```bash
awk '{print $2}' file
```

### Applicable Scenario

```text
Extract second field

View fixed position fields in the log

Process spaces separated text
```

---

## Scenario 3: Extract Multiple Columns

### Command

```bash
awk '{print $1,$2,$3}' file
```

### Explanation

Multiple columns are separated by spaces by default in the output.

### Example

Extracting the first three columns:

```bash
awk '{print $1,$2,$3}' access.log
```

---

## Scenario 4: Extract Entire Line

### Command

```bash
awk '{print $0}' file
```

### Explanation

```text
$0
→ Current whole line content
```

This is similar to outputting the file content directly, but often used with conditional judgments.

For example:

```bash
awk '$1 == "10.0.0.5" {print $0}' access.log
```

---

## Scenario 5: Extract Last Column

### Command

```bash
awk '{print $NF}' file
```

### Approach

```text
NF
→ Number of active line fields

$NF
→ Last row
```

### Applicable Scenario

```text
Ripping the last field of the log

Ripping time-consuming fields

Extract status field

Extract command output last column
```

---

## Scenario 6: Extract Penultimate Column

### Command

```bash
awk '{print $(NF-1)}' file
```

### Approach

```text
$(NF-1)
→ 2nd Last Column
```

### High-Frequency Memory

```text
Last row
→ $NF

2nd Last Column
→ $(NF-1)

Line
→ $0
```

---

## Scenario 7: Print Line Number and Content Together

### Command

```bash
awk '{print NR,$0}' file
```

### Explanation

```text
NR
→ Current Line Number

$0
→ Current whole line
```

### Applicable Scenario

```text
Add line numbers to logs

View text content locations

Auxiliary Positioning Problem Line
```

---

## Scenario 8: View Number of Fields per Line

### Command

```bash
awk '{print NF,$0}' file
```

### Function

View the number of columns per line and print the entire line content.

Suitable for judging:

```text
Complete Log Fields

Whether certain rows are formatted abnormally

Is the separator fit?
```

---

## Five, awk: Specifying Separator

---

## Scenario 9: Specify Colon as Separator

### Command

```bash
awk -F: '{print $1}' /etc/passwd
```

### Common Parameters

```text
-F
→ Specify Separator
```

### Explanation

`/etc/passwd` Uses colon `:` to separate fields.

Extracting the first column is the username.

---

## Scenario 10: Extract Username and UID from /etc/passwd

### Command

```bash
awk -F: '{print $1,$3}' /etc/passwd
```

Field explanation:

```text
$1
→ Username

$3
→ UID
```

---

## Scenario 11: Specify Comma as Separator

### Command

```bash
awk -F, '{print $1,$2}' file.csv
```

### Applicable Scenario

```text
Simple CSV Documentation

Comma Separator Log

Export statistical files
```

Note:

```text
Complex CSV when the field contains quotations and commas,awk It may not be accurate to split directly by comma.
```

---

## Scenario 12: Specify Multiple Separators

### Command

```bash
awk -F'[:,]' '{print $1,$2}' file
```

### Explanation

This command means splitting by colon and comma simultaneously.

Suitable for simple scenarios.

---

## Six, awk: Conditional Filtering

---

## Scenario 13: Conditional Filtering of Entire Line

### Command

```bash
awk '$1 == "10.0.0.5" {print $0}' access.log
```

### Approach

```text
If the first column equals 10.0.0.5

Print the whole line
```

### Applicable Scenario

```text
Filter Some IP Visit log

Filter a user

Filter a status field

Filter a hostname
```

---

## Scenario 14: Filter Specified Status Code

### Command

```bash
awk '$9 == 500 {print $0}' access.log
```

### Explanation

In common Nginx access.log:

```text
$9
→ HTTP Status Code
```

This command is used to filter requests with status code 500.

---

## Scenario 15: Multi-Condition Filtering

### Command

```bash
awk '$1 == "10.0.0.5" && $9 == 500 {print $0}' access.log
```

### Explanation

```text
&&
→ And...

||
→ Or...
```

### Meaning

```text
First column IP equals 10.0.0.5

And...

I don't think so. 9 Column status code equals 500

Print the whole line only
```

---

## Scenario 16: Or Condition Filtering

### Command

```bash
awk '$9 == 500 || $9 == 502 {print $0}' access.log
```

### Meaning

Outputs log lines with status code 500 or 502.

---

## Scenario 17: Numerical Greater Than Judgment

### Command

```bash
awk '$9 >= 500 {print $0}' access.log
```

### Applicable Scenario

```text
Filter 5xx Request

Filter requests that take longer than a certain threshold

Filter abnormal lines for numerical fields
```

---

## Scenario 18: Regular Expression Matching Entire Line

### Command

```bash
awk '$0 ~ /error/ {print $0}' app.log
```

### Explanation

```text
~
→ The right match.

$0
→ Line
```

This command represents:

```text
Include in line error, print the entire line
```

---

## Scenario 19: Regular Expression Matching Specified Column

### Command

```bash
awk '$7 ~ /^\/api/ {print $0}' access.log
```

### Meaning /think

If the 7th column starts with `/api`, print the entire line.

Suitable for:

```text
Filter API Request

Filter Fixed Paths

Filter a class URL
```

---

## 7. awk: Counting, Summing, and Calculating Averages

---

## Scenario 20: Counting the number of status codes equal to 500

### Command

```bash
awk '$9 == 500 {count++} END {print count}' access.log
```

### Logic

```text
Meet Status Code 500 Lines
→ count Add 1

After document processing
→ Output count
```

---

## Scenario 21: Counting the number of accesses from a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' access.log
```

### Applicable Scenarios

```text
Statistics a client IP Number of visits

Statistics of the occurrence of a user

Count the number of a status field
```

---

## Scenario 22: Counting requests in the 5xx range

### Command

```bash
awk '$9 >= 500 {count++} END {print count}' access.log
```

### Explanation

Filter lines where the 9th column is greater than or equal to 500 and count them.

---

## Scenario 23: Summing

### Command

```bash
awk '{sum+=$1} END {print sum}' file
```

### Explanation

Accumulate all numeric values in the first column.

### Applicable Scenarios

```text
Total request for statistics

Total statistical time-consuming

Total statistical flows

Statistics the sum of values for a given column
```

---

## Scenario 24: Calculating the average

### Command

```bash
awk '{sum+=$1; count++} END {if(count>0) print sum/count}' file
```

### Logic

```text
sum
→ Cumulative value

count
→ Lines

sum/count
→ Average
```

### Note

Here we added:

```text
if(count>0)
```

To avoid division by zero in empty files.

---

## Scenario 25: Counting occurrences of distinct values in a column

### Command

```bash
awk '{count[$1]++} END {for (item in count) print item,count[item]}' file
```

### Explanation

This is a basic usage of associative arrays in awk.

Suitable for counting:

```text
IP Number of occurrences

State code occurrence

Number of occurrences of user names

Number of interface path occurrences
```

However, in daily operations, the more commonly used combination command is:

```bash
awk '{print $1}' file | sort | uniq -c | sort -nr
```

---

## 8. awk: Passing Variables

---

## Scenario 26: Passing variables to awk

### Command

```bash
awk -v ip="10.0.0.5" '$1 == ip {print $0}' access.log
```

### Common Parameters

```text
-v
→ Here. awk Import Variables
```

### Explanation

`ip` is a variable passed to awk.

Suitable for use in scripts.

---

## Scenario 27: Passing variables in Shell

### Example

```bash
target_ip="10.0.0.5"
awk -v ip="$target_ip" '$1 == ip {print $0}' access.log
```

### Applicable Scenarios

```text
Dynamic specification in script IP

Dynamic Assign Status Code

Dynamic specify username

Dynamicly assign thresholds
```

---

## Scenario 28: Passing a status code variable

### Command

```bash
status_code=500
awk -v code="$status_code" '$9 == code {print $0}' access.log
```

---

## 9. Summary of Common awk Parameters and Variables

---

## 1. Common awk Parameters

```text
-F
→ Specify Separator

-v
→ Transfer Variables

-f
→ From awk Script File Read Rules
```

---

## 2. Common awk Variables

```text
$1
→ First column

$2
→ Column 2

$NF
→ Last row

$(NF-1)
→ 2nd Last Column

NF
→ Current line fields

NR
→ Current processed line number

$0
→ Current whole line content
```

---

## 3. Common awk Comparison Operators

```text
==
→ equals

!=
→ Not equal to

>
→ Greater than

>=
→ greater than or equal to

<
→ 小于

<=
→ 小于等于

&&
→ 并且

||
→ 或者

~
→ 正则匹配

!~
→ 正则不匹配
```

---

## 10. cut: Simple Column Extraction

---

## Scenario 29: Extracting the first column by colon

### Command

```bash
cut -d: -f1 /etc/passwd
```

### Common Parameters

```text
-d
→ 指定分隔符

-f
→ 指定字段

-c
→ 按字符位置截取
```

### Explanation

`cut` is suitable for simple text with fixed delimiters.

For example, files separated by colons like `/etc/passwd`.

---

## Scenario 30: Extracting multiple columns by colon

### Command

```bash
cut -d: -f1,3 /etc/passwd
```

### Meaning

Output the 1st and 3rd columns.

Which is:

```text
用户名

UID
```

---

## Scenario 31: Extracting by character

### Command

```bash
cut -c1-10 file
```

### Explanation

```text
-c1-10
→ 截取每行第 1 到第 10 个字符
```

### Applicable Scenarios

```text
固定长度文本

截取日志时间前缀

截取固定字段位置
```

---

## Scenario 32: Choosing between cut and awk

cut is suitable for:

```text
简单分隔符

固定字段

只取列，不做复杂判断
```

awk is suitable for:

```text
复杂条件判断

字段计算

多个分隔符

按列过滤

统计计算
```

Simple example:

```bash
cut -d: -f1 /etc/passwd
```

For more complex cases, it's recommended to use:

```bash
awk -F: '$3 >= 1000 {print $1,$3}' /etc/passwd
```

---

## 11. sort: Sorting

---

## Scenario 33: Normal sorting

### Command

```bash
sort file
```

### Explanation

Default sorting by dictionary order.

---

## Scenario 34: Sorting by numeric value

### Command

```bash
sort -n file
```

### Explanation

```text
-n
→ Sort by Number Size
```

Suitable for numeric lists.

---

## Scenario 35: Reverse sorting

### Command

```bash
sort -r file
```

### Explanation

```text
-r
→ Rewind
```

---

## Scenario 36: Reverse numeric sorting

### Command

```bash
sort -nr file
```

### Explanation

```text
-n
→ Number Sorting

-r
→ Rewind
```

Commonly used for sorting TopN statistics results.

---

## Scenario 37: Sorting by the second column

### Command

```bash
sort -k2 file
```

### Explanation

```text
-k2
→ Sort by Second Column
```

---

## Scenario 38: Reverse numeric sorting by the second column

### Command

```bash
sort -k2 -nr file
```

### Applicable Scenarios

```text
Sort by number of statistics

Sort by Time

Sort by Size

Sort by second column values
```

---

## Scenario 39: Sorting and deduplication

### Command

```bash
sort -u file
```

### Explanation

```text
-u
→ Sort After Heavy
```

---

## 12. Summary of Common sort Parameters

```text
-n
→ Sort by Number

-r
→ Rewind

-u
→ Sort After Heavy

-k
→ Sort by specified column

-h
→ Sort by human readable size, for example KI don't know.MI don't know.G
```

Common commands:

```bash
sort file
```

```bash
sort -n file
```

```bash
sort -nr file
```

```bash
sort -k2 file
```

```bash
sort -k2 -nr file
```

```bash
sort -u file
```

---

## 13. uniq: Deduplication and Counting Duplicates

---

## Scenario 40: Deduplication

### Command

```bash
sort file | uniq
```

### Explanation

Usually used with `sort`.

Reason:

```text
uniq Only handle next-door repeat lines
```

---

## Scenario 41: Counting duplicate occurrences

### Command

```bash
sort file | uniq -c
```

### Common Parameters

```text
-c
→ Number of statistical repetitions

-d
→ Show repeat lines only

-u
→ Show only one row
```

---

## Scenario 42: Counting TopN duplicate content

### Command

```bash
sort file | uniq -c | sort -nr | head
```

### Logic

```text
sort file
→ Sort first

uniq -c
→ Number of statistical repetitions

sort -nr
→ In descending order of numbers

head
→ Before 10 First Name
```

---

## Scenario 43: Displaying only duplicate lines

### Command

```bash
sort file | uniq -d
```

### Explanation

```text
-d
→ Show only duplicated lines
```

---

## Scenario 44: Displaying only unique lines

### Command

```bash
sort file | uniq -u
```

### Explanation

```text
-u
→ Show only lines that do not repeat
```

---

## 14. Summary of Common uniq Parameters

```text
-c
→ Number of statistical repetitions

-d
→ Show repeat lines only

-u
→ Show only one row
```

Common commands:

```bash
sort file | uniq
```

```bash
sort file | uniq -c
```

```bash
sort file | uniq -c | sort -nr | head
```

```bash
sort file | uniq -d
```

```bash
sort file | uniq -u
```

---

## 15. wc: Counting

---

## Scenario 45: Counting lines

### Command

```bash
wc -l file
```

### Explanation

```text
-l
→ Lines
```

### Applicable Scenarios

```text
Total number of statistical log lines

Number of statistical filter results

Number of statistical files

Number of statistical errors
```

---

## Scenario 46: Counting words

### Command

```bash
wc -w file
```

### Explanation

```text
-w
→ Number of words
```

---

## Scenario 47: Counting bytes

### Command

```bash
wc -c file
```

### Explanation

```text
-c
→ Bytes
```

---

## Scenario 48: Counting filtered lines

### Command

```bash
grep "error" app.log | wc -l
```

### Explanation

```text
grep Filter error

wc -l Number of statistical lines
```

---

## 16. Summary of Common wc Parameters

```text
-l
→ Lines

-w
→ Number of words

-c
→ Bytes
```

Common commands:

```bash
wc -l file
```

```bash
wc -w file
```

```bash
wc -c file
```

```bash
grep "error" app.log | wc -l
```

---

## 17. Common Combined Patterns

---

## Scenario 49: General pattern for TopN statistics

### Command /think

```bash
awk '{print Column}' file | sort | uniq -c | sort -nr | head
```

### Applicable Scenarios

```text
Most visits IP

Most visits URL

Repeat error TopN

Status Code Distribution

User Name Statistics

Hostname statistics
```

---

## Scenario 50: Count the Most Visited IP

### Command

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

### Approach

```text
awk '{print $1}'
→ Rip first column IP

sort
→ Sort

uniq -c
→ Number of statistical repetitions

sort -nr
→ In descending order

head
→ Before 10 First Name
```

---

## Scenario 51: Count Status Code Distribution

### Command

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

### Explanation

Common Nginx access.log entries:

```text
$9
→ Status Code
```

Output similar to:

```text
1000 200
50 404
20 500
```

---

## Scenario 52: Count the Most Visited URL

### Command

```bash
awk '{print $7}' access.log | sort | uniq -c | sort -nr | head
```

### Explanation

Common Nginx access.log entries:

```text
$7
→ URL Path
```

---

## Scenario 53: Count How Many Times a URL is Accessed

### Command

```bash
awk '{print $7}' access.log | grep "^/api/login$" | wc -l
```

### Explanation

```text
awk '{print $7}'
→ Extract URL

grep "^/api/login$"
→ Exact Match /api/login

wc -l
→ Number of statistics
```

---

## Scenario 54: Count 5xx Errors

### Command

```bash
awk '$9 >= 500 {print $0}' access.log | wc -l
```

Or:

```bash
awk '$9 >= 500 {count++} END {print count}' access.log
```

---

## Scenario 55: Filter 500 Status Code Requests

### Command

```bash
awk '$9 == 500 {print $0}' access.log
```

---

## Scenario 56: Count and Sort by Status Code

### Command

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

---

## Scenario 57: Count Status Code Distribution for a Specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $9}' access.log | sort | uniq -c | sort -nr
```

### Applicable Scenarios

```text
Analyse whether a certain client is large 4xx / 5xx

Analysis of a source IP Result of request

Check a user access anomaly
```

---

## Eighteen. Production Troubleshooting Notes

---

## 1. First Confirm Field Positions

Field positions may vary across different log formats.

For example, in Nginx access.log:

```text
$1
→ IP

$7
→ URL

$9
→ Status Code
```

But this depends on `log_format`.

So before formal statistics, check a few lines:

```bash
head -n 5 access.log
```

Or:

```bash
tail -n 5 access.log
```

---

## 2. Sort + Uniq Requires Pre-Sorting

Do not directly use:

```bash
uniq -c file
```

More commonly:

```bash
sort file | uniq -c
```

Reason:

```text
uniq Only count the adjacent repeat lines
```

---

## 3. awk Defaults to Space-Separated Fields

If the file uses colon, comma, or pipe as separators, specify the delimiter.

Colon:

```bash
awk -F: '{print $1}' /etc/passwd
```

Comma:

```bash
awk -F, '{print $1}' file.csv
```

Pipe:

```bash
awk -F'|' '{print $1}' file
```

---

## 4. Be Cautious with Fields Containing Spaces

For example, Nginx request line:

```text
"GET /api/login HTTP/1.1"
```

Contains spaces.

If the log format is complex, field positions may differ from expectations.

Confirm the log format before troubleshooting.

---

## 5. Statistics Should Combine Time Ranges

For example:

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

Counts the entire file.

If the log spans multiple days, it cannot represent current failures directly.

Later, combine with time fields, grep, or awk time filtering to narrow the scope.

---

## Nineteen. Summary of Common Commands in This Article

---

## awk Column Extraction

```bash
awk '{print $1}' file
```

```bash
awk '{print $2}' file
```

```bash
awk '{print $1,$2,$3}' file
```

```bash
awk '{print $0}' file
```

```bash
awk '{print $NF}' file
```

```bash
awk '{print $(NF-1)}' file
```

```bash
awk '{print NR,$0}' file
```

```bash
awk '{print NF,$0}' file
```

---

## awk Specify Separator

```bash
awk -F: '{print $1}' /etc/passwd
```

```bash
awk -F: '{print $1,$3}' /etc/passwd
```

```bash
awk -F, '{print $1,$2}' file.csv
```

```bash
awk -F'[:,]' '{print $1,$2}' file
```

---

## awk Conditional Filtering

```bash
awk '$1 == "10.0.0.5" {print $0}' access.log
```

```bash
awk '$9 == 500 {print $0}' access.log
```

```bash
awk '$1 == "10.0.0.5" && $9 == 500 {print $0}' access.log
```

```bash
awk '$9 == 500 || $9 == 502 {print $0}' access.log
```

```bash
awk '$9 >= 500 {print $0}' access.log
```

```bash
awk '$0 ~ /error/ {print $0}' app.log
```

```bash
awk '$7 ~ /^\/api/ {print $0}' access.log
```

---

## awk Statistics

```bash
awk '$9 == 500 {count++} END {print count}' access.log
```

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' access.log
```

```bash
awk '$9 >= 500 {count++} END {print count}' access.log
```

```bash
awk '{sum+=$1} END {print sum}' file
```

```bash
awk '{sum+=$1; count++} END {if(count>0) print sum/count}' file
```

```bash
awk '{count[$1]++} END {for (item in count) print item,count[item]}' file
```

---

## awk Pass Variables

```bash
awk -v ip="10.0.0.5" '$1 == ip {print $0}' access.log
```

```bash
target_ip="10.0.0.5"
awk -v ip="$target_ip" '$1 == ip {print $0}' access.log
```

```bash
status_code=500
awk -v code="$status_code" '$9 == code {print $0}' access.log
```

---

## cut

```bash
cut -d: -f1 /etc/passwd
```

```bash
cut -d: -f1,3 /etc/passwd
```

```bash
cut -c1-10 file
```

---

## sort

```bash
sort file
```

```bash
sort -n file
```

```bash
sort -r file
```

```bash
sort -nr file
```

```bash
sort -k2 file
```

```bash
sort -k2 -nr file
```

```bash
sort -u file
```

---

## uniq

```bash
sort file | uniq
```

```bash
sort file | uniq -c
```

```bash
sort file | uniq -c | sort -nr | head
```

```bash
sort file | uniq -d
```

```bash
sort file | uniq -u
```

---

## wc

```bash
wc -l file
```

```bash
wc -w file
```

```bash
wc -c file
```

```bash
grep "error" app.log | wc -l
```

---

## Common Combinations

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

```bash
awk '{print $7}' access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $7}' access.log | grep "^/api/login$" | wc -l
```

```bash
awk '$9 >= 500 {print $0}' access.log | wc -l
```

```bash
awk '$1 == "10.0.0.5" {print $9}' access.log | sort | uniq -c | sort -nr
```

---

## Twenty. One-Sentence Summary

The coreDivision of labour of awk, cut, sort, uniq, and wc is:

```text
awk
→ Complex field extraction, condition filtering, statistical calculation

cut
→ Simple fixed separator row

sort
→ Sort

uniq
→ We're going to repeat the statistics.

wc
→ Count
```

The core pattern for log statistics is:

```text
Extract fields first

→ Reorder

→ We'll go over the statistics.

→ Rewind.

→ Last taken TopN
```

Most commonly used combination:

```bash
awk '{print Column}' file | sort | uniq -c | sort -nr | head
```

Common scenarios:

```text
Most statistical visits IP

Statistical status code distribution

Most statistical visits URL

Count a certain IP Request status code distribution

Count the occurrence of a key field
```

Production recommendations:

```text
Confirm log field position before statistics

uniq It's usually first. sort

Complex Separator Prefer awk

Simple fixed separator can be used cut

The results need to be combined with time frames.

Do not directly package field numbers when log format is not known
```