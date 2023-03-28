# BigQuery 固定宽度比较

> 原文：<https://medium.com/google-cloud/bigquery-fixed-width-comparison-958b1fd02b37?source=collection_archive---------1----------------------->

又是一天，又是一个挑战—一位客户想要解析和比较固定宽度的文本文件。这些文件经常被大型机使用，并用于数据交换。这是为了用遗留管道验证一些新的基于 BigQuery 的管道。

这些文件是-

*   换行符分隔(就像普通的 Unix/Mac 文本文件一样)
*   分成固定数量字符的字段
*   根据需要用空格填充数字和字符串，以形成宽度
*   可以有几个不同的记录类型(它可以非常依赖于一个众所周知的记录类型字段)

***可以用 BigQuery 解决这个吗？是的。我会告诉你怎么做。***

# 概观

在这篇博文中，我们将-

*   根据 GCS 中的固定宽度文件创建一个外部 CSV 表
*   使用配置表— *它也可以在 GCS 中*
*   创建一个视图来提取要比较的字段
*   与帮助函数进行比较

# 题外话:UTF-8 对 ISO-8859–1

BigQuery 有两种用于解释 CSV 文件的字符编码:UTF-8 和 ISO-8859–1。许多规范不会提到字符编码，因此了解它们的历史会很有用。

*注意:这是作者的节略摘要——可能会遗漏一些细节，但这会让你更好地理解任何进一步的研究。*

## ASCII: 8 位字符，7 位标准

ASCII [以 7 位(128)字符集开始](https://en.wikipedia.org/wiki/ASCII#Bit_width)，但使用 8 位(一个字节，或 256 字符集)编码。这留下了用于奇偶校验或其他目的的备用位。*如果所有字符都使用这个字符子集，那么 ISO-8859–1 和 UTF-8 是相同的。*

## ISO-8859–1

ISO-8859–1 将任何剩余高位(剩余 128 个字符)解释为某个[扩展字符集](https://en.wikipedia.org/wiki/ISO/IEC_8859-1)。无论如何，每个字节都是一个字符。这使得查找第 N 个字符变得容易得多——只需转到第 N 个字节。

*除非规范要求 UTF-8，否则我建议使用 ISO-8859–1。固定宽度文件格式的时代早于 UTF-8 的普及。*

## UTF-8

另一方面，UTF-8 是当今大多数环境(浏览器、操作系统、语言等)使用的最成熟的 Unicode 标准。字符有**可变长度**，所以它们可能是一个字节、两个字节或更多。高位(那 128 个字符)用于确定**字符中还有多少字节**。这些更多的字节允许编码任何多种 unicode 字符。这使得查找第 N 个字符变得更加困难——您需要解释字节来查找它。

# 创建外部表

BigQuery 脚本从固定宽度文件的映射开始。这使用了 [CSV 外部表](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv)格式，并利用了几乎所有固定宽度文件都有他们从不使用的字符这一事实——这将是我们的 CSV 分隔符。

所有行都有一个字符串作为行。

```
CREATE OR REPLACE EXTERNAL TABLE temp.utf8_files (row STRING)
OPTIONS (
  -- Standard options
  allow_jagged_rows = false,
  allow_quoted_newlines = false,
  format = CSV,
  max_bad_records = 0,
  ignore_unknown_values = false,
  -- UTF8 or ISO-8859-1
  -- ISO-8859-1 interprets the high-bits as single-byte,
  -- while UTF-8 allows for multi-byte characters
  encoding = UTF8,
  -- Ensure this is never used in the fixed-width file.
  -- This can be an ISO-8859-1 character.
  field_delimiter = '*',
  -- Skip any headers (sometimes necessary)
  skip_leading_rows = 1,
  -- Source of files in GSC
  -- TEST refers to the feed type
  uris = [
    'gs://<bucket>/fixed-width/utf8/TEST/*'
  ]
);
```

# 创建配置表

因为您可能有许多不同的提要，所以在配置表中将其外部化会很有用。这个表可以是 GCS 中的一个外部表，或者在这种情况下，它只是用 SQL 来表示，以帮助测试。

配置字段-

*   **进给 _ 类型。**这是配置名称(如何解释文件)。
*   **记录开始，记录长度。**记录类型的起始位置和长度
*   **直肠型。**这是匹配的记录类型。如果等于这个值，则配置的其余部分适用
*   **关键 _ 开始，关键 _ 长度。**比较键的起始位置和长度。该键必须是唯一的，即主键。
*   **行 _ 开始，行 _ 长度。**要比较的行的起始位置和长度

## 配置 SQL

```
CREATE OR REPLACE TABLE temp.config_data AS (
  SELECT * FROM UNNEST([
    STRUCT(
        'TEST' AS feed_type,
        1 AS rectype_start, 3 AS rectype_length,
        'R00' AS rectype,
        72 AS key_start, 50 AS key_length,
        1 AS row_start, 1000 AS row_length),
     STRUCT(
        'TEST' AS feed_type,
        1 AS rectype_start, 3 AS rectype_length,
        'R01' AS rectype,
        72 AS key_start, 50 AS key_length,
        1 AS row_start, 1000 AS row_length),
    STRUCT(
        'TEST' AS feed_type,
        1 AS rectype_start, 3 AS rectype_length,
        'R02' AS rectype,
        72 AS key_start, 50 AS key_length,
        1 AS row_start, 1000 AS row_length),
  ])
);
```

# 创建固定宽度文件的视图

既然 BigQuery 中提供了固定宽度的文件，那么以通用的方式向消费者提供数据将使将来的比较更加容易。

请注意，配置提要名称是从 GCS 目录名称中推断出来的。这允许您添加新的配置，只需将数据复制到正确的目录中，视图就可以工作了。

该视图将返回以下字段。在应用了配置信息后，所有字段都是*。*

*   fname。行的文件名(在目录之后)。
*   **饲料 _ 类型。**饲料的种类。
*   **直肠型。**记录类型。
*   **键。**唯一的比较键。
*   **排。**比较值。

```
CREATE OR REPLACE VIEW temp.external_files AS
WITH
  SourceData AS (
    SELECT
      REGEXP_EXTRACT(_FILE_NAME,
        'gs://<bucket>/fixed-width/utf8/[^/]+/(.*)') AS fname,
      REGEXP_EXTRACT(_FILE_NAME,
        'gs://<bucket>/fixed-width/utf8/([^/]+)') AS feed_type,
      row
    FROM
      temp.utf8_files
  )
SELECT
  fname,
  s.feed_type,
  SUBSTRING(row, c.rectype_start, c.rectype_length) AS rectype,
  SUBSTRING(row, c.key_start, c.key_length) AS key,
  SUBSTRING(row, c.row_start, c.row_length) AS row
FROM
  SourceData s
  LEFT JOIN temp.config_data c ON (
    c.feed_type=s.feed_type AND
    SUBSTRING(row, c.rectype_start, c.rectype_length) = c.rectype
  );
```

# 做一下比较——差不多！

比较两个不同的文件应该是容易的，使用用户定义的函数，这可以变得容易。

## 要比较的 SQL(文件 1，文件 2)

```
SELECT
  rectype,
  key,
  CompareSet(ARRAY_AGG(f)) AS status
FROM
  temp.external_files f
WHERE
  fname IN (
    'file1',
    'file2'
  )
GROUP BY
  rectype, key
HAVING
  status IS NOT NULL;
```

如果就这些就好了！

在这种情况下，CompareSet 从 external_files 接收结构数组或行数组。如果行之间没有差异，则返回 NULL。

上面的注释—*big query 只读取文件 1 和文件 2。作为外部表，它将是高效的。乐观主义者会为你工作得很好。*

## 对比显示

CompareSet 将首先检查行数是否不是 2(什么？)或者，如果两个，来自同一个文件(什么？).如果出现这种情况，那么它只报告文件和奇数个键。如果键不是唯一的，有两个以上的文件，或者一方或另一方缺少数据，就会发生这种情况。

如果行不同，那么我们调用另一个函数— CodePointDiff —来报告字符差异(列，特定的列差异)。注意 **TO_CODE_POINTS** 功能。由于 BigQuery 使用了 **UTF-8** ，为了遍历(数组中的)字符，它必须被转换。参见**题外话**获得解释。

否则它就是空的——一切都好！它们匹配。

```
CREATE TEMP FUNCTION CompareSet(s ANY TYPE) AS (
  CASE
    -- If there isn't two rows OR two rows are the same filename,
    -- then report this as a structural issue.
    WHEN ARRAY_LENGTH(s) != 2 OR
         s[SAFE_OFFSET(0)].fname = s[SAFE_OFFSET(1)].fname THEN
      -- Report the unique set of row counts
      (
        SELECT
          STRING_AGG(CONCAT(fname, ' has ', count, ' row(s)'), ', ')
        FROM (
          SELECT
            fname,
            COUNT(*) AS count
          FROM
            UNNEST(s)
          GROUP BY
            fname
        )
      )

    -- Row content are different. Return where the difference is.
    WHEN s[SAFE_OFFSET(0)].row != s[SAFE_OFFSET(1)].row THEN
      CodePointDiff(
        s[SAFE_OFFSET(0)].fname,
        TO_CODE_POINTS(s[SAFE_OFFSET(0)].row),
        s[SAFE_OFFSET(1)].fname,
        TO_CODE_POINTS(s[SAFE_OFFSET(1)].row)) -- Two items in the array, different filenames, same row.
    -- No difference! All good.
    ELSE
       NULL
  END
);
```

## CodePointDiff 显示

这将报告**字符**的位置。对于较低的 7 位字符(通常出现在固定宽度的文件中)，这将对应于字节数。对于多字节 UTF-8 字符，它不会。

该函数将首先检查数组长度，然后找到行中第一个不同的字符，并将其与字符一起报告。

*注意，使用 SAFE_OFFSET()可能很重要，因为有时 BigQuery 会执行 SQL，即使它不需要执行。*

```
-- Compare an array of codepoints for two strings. This is valid
-- only where the source STRINGs are different.
CREATE TEMP FUNCTION CodePointDiff(
    lname STRING, l ARRAY<INT64>,
    rname STRING, r ARRAY<INT64>) AS (
  IF(
    ARRAY_LENGTH(l) != ARRAY_LENGTH(r),
    CONCAT("Length of ", lname, " is ", ARRAY_LENGTH(l),
           ", ", rname, " ", ARRAY_LENGTH(r)),
    (SELECT
       CONCAT("Char ", idx+1,
         " is different ",
         lname,
         "='",
         CODE_POINTS_TO_STRING([ch]),
         "' vs ",
         rname,
         "='",
         CODE_POINTS_TO_STRING([ r[SAFE_OFFSET(idx)]]),
         "'"
       )
     FROM
       UNNEST(l) AS ch
       WITH OFFSET idx
     WHERE
       ch != r[SAFE_OFFSET(idx)]
     LIMIT 1
    )
  ));
```

# 结论

这是一个比较固定宽度文件的 BigQuery 世界之旅，但希望对演示一些有趣的 BigQuery 技术有用！