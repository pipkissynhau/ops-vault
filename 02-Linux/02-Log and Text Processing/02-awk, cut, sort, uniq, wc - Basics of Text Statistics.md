# 02-awk, cut, sort, uniq, wc: Basics of Text Statistics

# Linux # Log Analysis # Text Processing # awk # cut # sort # uniq # wc # Field Extraction # Statistical Analysis # Operations and Maintenance

---

## Recommended Path

01-Linux Basics and Server Operations/02-Logs and Text Processing/02-awk, cut, sort, uniq, wc: Basics of Text Statistics.md

---

## I. Document Overview

This article compiles commonly used **awk, cut, sort, uniq, wc** commands in Linux log analysis and text processing.

Key topics include:

- Using awk to extract fields by column
- Extracting the first, multiple, last, or second-to-last column with awk
- Specifying delimiters in awk
- Conditional filtering with awk
- Multiple conditional judgments in awk
- Regular expression matching with awk
- Counting, summing, and averaging values with awk
- Passing variables to awk
- Simple field extraction with cut
- Sorting data with sort
- Removing duplicates and counting occurrences with uniq
- Counting lines, words, and bytes with wc
- Common combinations of sort + uniq + wc
- General approaches for TopN text statistics

This article is the 02nd in the log and text processing series, aimed at helping users:

```text
How to extract fields, apply filters, count values, and sort data from logs or texts
```

The goal is to enable users to:

- Understand the structure of text fields
- Use awk to extract specified columns
- Filter logs based on conditions with awk
- Perform basic statistical analysis using awk
- Simplely split fields with cut
- Sort and remove duplicates using sort and uniq
- Conduct basic counting tasks with wc
- Combine commands for common log statistical analyses

---

## II. Command Overview

The commands discussed in this article include:

```text
awk
→ Extracting fields by column, performing conditional judgments, and calculating statistics
cut
→ Simple field extraction, suitable for texts with fixed delimiters
sort
→ Sorting data
uniq
→ Removing duplicates and counting occurrences
wc
→ Counting lines, words, and bytes
```

In summary:

```text
awk is ideal for handling complex fields
cut is better for simple field splitting
sort is used for sorting data
uniq removes duplicates and provides count information
wc calculates basic statistics
```

---

## III. Basic Concepts

---

## 1. Columns in awk

By default, awk splits fields based on blank characters.

Example text:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123
```

In awk:

```text
$1
→ First column
$2
→ Second column
$3
→ Third column
$0
→ Entire line
$NF
→ Last column
$(NF-1)
→ Second-to-last column
```

Example:

```bash
awk '{print $1}' access.log
```

Meaning:

```text
Print the first column of access.log
```

---

## 2. NF and NR in awk

### NF

`NF` indicates the total number of columns in the current line.

Example:

```bash
awk '{print NF}' file
```

Meaning:

```text
Print the number of fields in each line
```

### NR

`NR` represents the current line number.

Example:

```bash
awk '{print NR,$0}' file
```

Meaning:

```text
Print the line number and the entire line content
```

---

## 3. The relationship between sort and uniq

`uniq` can only handle adjacent duplicate rows.

Therefore, it is generally necessary to first use `sort` and then `uniq`.

Common usage:

```bash
sort file | uniq -c
```

Meaning:

```text
First, sort the data so that identical items are grouped together.
Then, use uniq -c to count the number of occurrences for each group.
```

---

## 4. General approaches for text statistics

The most common statistical routine is:

```bash
awk '{print 某列}' file | sort | uniq -c | sort -nr | head
```

Meaning:

```text
awk
→ Extract the desired field
sort
→ Sort the data so that identical items are adjacent
uniq -c
→ Count the number of occurrences for each unique value
sort -nr
→ Sort the results in descending order based on the count
head
→ Display the top few entries
```

Applicable scenarios:

```text
Counting the most frequently visited IPs
Identifying the most common URLs
Analyzing HTTP status codes distribution
Determining the error keywords with the highest frequency
Finding```bash
awk '$7 ~ /^\/api/ {print $0}' access.log
```### Explanation

```text
-c
→ Number of bytes
```

---

## Scenario 48: Counting the number of lines after filtering

### Command

```bash
grep "error" app.log | wc -l
```

### Explanation

```text
grep filters for "error"

wc -l counts the number of lines
```

---

## XVI. Summary of Common Parameters for wc

```text
-l
→ Number of lines

-w
→ Number of words

-c
→ Number of bytes
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

## XVII. Common Combination Patterns

---

## Scenario 49: General pattern for counting TopN items

### Command

```bash
awk '{print column}' file | sort | uniq -c | sort -nr | head
```

### Application Scenarios

```text
Top N most frequently accessed IPs

Top N most frequently visited URLs

Top N repeated errors

Distribution of status codes

Statistics for usernames

Statistics for hostnames
```

---

## Scenario 50: Counting the most frequently accessed IP

### Command

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

### Approach

```text
awk '{print $1}'
→ Extracts the first column, which is the IP address

sort
→ Sorts the list

uniq -c
→ Counts the number of unique IPs

sort -nr
→ Arranges the results in descending order based on frequency

head
→ Displays the top 10 entries
```

---

## Scenario 51: Counting the distribution of status codes

### Command

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

### Explanation

In a typical Nginx access.log file:

```text
$9
→ Status code
```

The output will look something like this:

```text
1000 200
50 404
20 500
```

---

## Scenario 52: Counting the most visited URL

### Command

```bash
awk '{print $7}' access.log | sort | uniq -c | sort -nr | head
```

### Explanation

In a typical Nginx access.log file:

```text
$7
→ URL path
```

---

## Scenario 53: Counting the number of visits to a specific URL

### Command

```bash
awk '{print $7}' access.log | grep "^/api/login$" | wc -l
```

### Explanation

```text
awk '{print $7}'
→ Extracts the URL path

grep "^/api/login$
→ Exactly matches the URL "/api/login"

wc -l
→ Counts the number of visits
```

---

## Scenario 54: Counting the number of 5xx status codes

### Command

```bash
awk '$9 >= 500 {print $0}' access.log | wc -l
```

or:

```bash
awk '$9 >= 500 {count++} END {print count}' access.log
```

---

## Scenario 55: Filtering out requests with status code 500

### Command

```bash
awk '$9 == 500 {print $0}' access.log
```

---

## Scenario 56: Counting and sorting by status code

### Command

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

---

## Scenario 57: Counting the distribution of request status codes for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $9}' access.log | sort | uniq -c | sort -nr
```

### Application Scenarios

```text
Analyzing whether a particular client experiences many 4xx or 5xx errors

Examining the request results for a specific source IP

Troubleshooting abnormal visits from a certain user
```

---

## XVIII. Notes for Production Troubleshooting

---

## 1. Confirm the field positions first

The field positions may vary in different log formats.

For example, in a typical Nginx access.log file:

```text
$1
→ IP address

$7
→ URL path

$9
→ Status code
```

However, this depends on the `log_format` setting. Therefore, before starting any statistical analysis, check the first few lines of the log file using:

```bash
head -n 5 access.log
```

or:

```bash
tail -n```bash
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

## 常用组合

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

## 二十、一句话总结

awk、cut、sort、uniq、wc 的核心分工是：

```text
awk
→ 复杂字段提取、条件过滤、统计计算

cut
→ 简单固定分隔符切列

sort
→ 排序

uniq
→ 去重和统计重复

wc
→ 计数
```

日志统计的核心套路是：

```text
先提取字段

→ 再排序

→ 再去重统计

→ 再倒序

→ 最后取 TopN
```

最常用组合：

```bash
awk '{print 某列}' file | sort | uniq -c | sort -nr | head
```

常见场景：

```text
统计访问最多 IP

统计状态码分布

统计访问最多 URL

统计某个 IP 请求状态码分布

统计某个关键字段出现次数
```

生产建议：

```text
统计前先确认日志字段位置

uniq 前通常要先 sort

复杂分隔符优先用 awk

简单固定分隔符可以用 cut

统计结果要结合时间范围

不要在不了解日志格式时直接套字段编号
```