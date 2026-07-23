--https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/quickstart-for-writing-on-github

<details>
<summary>oem_tablespace_report1</summary>
  
```SQL
set lines 220 pages 100
col "Available Space Used (%)" for a25
col "Allocated Space Used (%)" for a25
col "Allocated Free Space (GB)" for a25
col "Allocated Size (GB)" for a15
col "Space Used GB" for a15
col tablespace_name for a30
col AUTOEXTEND for a6
col status for a8
col CNT for a99999
WITH df AS (
    SELECT
        tablespace_name,
        SUM(bytes) bytes,
        COUNT(*) cnt,
        decode(SUM(decode(autoextensible, 'NO', 0, 1)), 0, 'NO', 'YES') autoext
    FROM
        dba_data_files
    GROUP BY
        tablespace_name
), um AS (
    SELECT
        tablespace_name,
        used_space ub,
        used_percent
    FROM
        dba_tablespace_usage_metrics
)
SELECT
    d.tablespace_name,
    to_char(u.used_percent, '99999990.00') "Available Space Used (%)",
    to_char(nvl((a.bytes - nvl(f.bytes, 0)) / a.bytes * 100, 0), '99999990.00') "Allocated Space Used (%)",
    a.autoext Autoextend,
    to_char(nvl(a.bytes, 0) / 1024 / 1024 / 1024, '99999990.000') "Allocated Size (GB)",
    to_char(nvl(a.bytes - nvl(f.bytes, 0), 0) / 1024 / 1024 / 1024, '99999990.000') "Space Used GB",
    to_char(nvl(f.bytes, 0) / 1024 / 1024 / 1024, '99999990.000') "Allocated Free Space (GB)",
    d.status,
    a.cnt,
    d.contents,
    d.extent_management,
    d.segment_space_management SSM
FROM
    dba_tablespaces d,
    df a,
    um u,
    (
        SELECT
            ts.name tablespace_name,
            nvl(SUM(e.blocks * ts.blocksize), 0) bytes
        FROM
            dba_lmt_free_space   e,
            sys.ts$              ts,
            dba_data_files       df
        WHERE
            ts.name = df.tablespace_name
            AND ts.ts# = e.tablespace_id
            AND e.file_id = df.relative_fno
        GROUP BY
            ts.name
        UNION ALL
        SELECT
            ts.name tablespace_name,
            nvl(SUM(fs.blocks * ts.blocksize), 0) bytes
        FROM
            dba_dmt_free_space   fs,
            sys.ts$              ts,
            dba_data_files       df
        WHERE
            ts.name = df.tablespace_name
            AND ts.ts# = fs.tablespace_id
            AND fs.file_id = df.relative_fno
        GROUP BY
            ts.name
    ) f
WHERE
    d.tablespace_name = a.tablespace_name (+)
    AND d.tablespace_name = f.tablespace_name (+)
    AND d.tablespace_name = u.tablespace_name (+)
    AND NOT d.contents = 'UNDO'
    AND NOT ( d.extent_management = 'LOCAL'
              AND d.contents = 'TEMPORARY' )
--    AND d.tablespace_name LIKE :1
UNION ALL
SELECT
    d.tablespace_name,
    to_char(u.used_percent, '99999990.00') "Available Space Used (%)",
    to_char(nvl((u.ub * d.block_size) / tf.bytes * 100, 0), '99999990.00') "Allocated Space Used (%)",
    tf.autoext Autoextend,
    to_char(nvl(tf.bytes, 0) / 1024 / 1024 / 1024, '99999990.000') "Allocated Size (GB)",
    to_char(nvl(u.ub * d.block_size, 0) / 1024 / 1024 / 1024, '99999990.000') "Space Used GB",
    to_char(( nvl(tf.bytes, 0) - nvl(u.ub * d.block_size, 0) ) / 1024 / 1024 / 1024, '99999990.000') "Allocated Free Space (GB)",
    d.status,
    tf.cnt,
    d.contents,
    d.extent_management,
    d.segment_space_management SSM
FROM
    dba_tablespaces                                                                                                                                                d,
    um                                                                                                                                                             u,
    (
        SELECT
            tablespace_name,
            SUM(bytes) bytes,
            COUNT(*) cnt,
            decode(SUM(decode(autoextensible, 'NO', 0, 1)), 0, 'NO', 'YES') autoext
        FROM
            dba_temp_files
        GROUP BY
            tablespace_name
    ) tf
WHERE
    d.tablespace_name = tf.tablespace_name (+)
    AND d.tablespace_name = u.tablespace_name (+)
    AND d.extent_management = 'LOCAL'
    AND d.contents = 'TEMPORARY'
--    AND d.tablespace_name LIKE :2
UNION ALL
SELECT
    d.tablespace_name,
    to_char(u.used_percent, '99999990.00') "Available Space Used (%)",
    to_char(nvl((u.ub * d.block_size) / a.bytes * 100, 0), '99999990.00') "Allocated Space Used (%)",
    a.autoext Autoextend,
    to_char(nvl(a.bytes, 0) / 1024 / 1024 / 1024, '99999990.000') "Allocated Size (GB)",
    to_char(nvl(u.ub * d.block_size, 0) / 1024 / 1024 / 1024, '99999990.000') "Space Used GB",
    to_char(( nvl(a.bytes, 0) - nvl(u.ub * d.block_size, 0) ) / 1024 / 1024 / 1024, '99999990.000') "Allocated Free Space (GB)",
    d.status,
    a.cnt,
    d.contents,
    d.extent_management,
    d.segment_space_management SSM
FROM
    dba_tablespaces   d,
    df                a,
    um                u
WHERE
    d.tablespace_name = a.tablespace_name (+)
    AND d.tablespace_name = u.tablespace_name (+)
    AND d.contents = 'UNDO'
--    AND d.tablespace_name LIKE :3
ORDER BY
    3 ASC;
```
</details>


<details>
<summary>List files of tablespace</summary>
  
```SQL
col file_name for a100
col TABLESPACE_NAME for a20
set lines 230 pages 70
WITH allfiles
     AS (SELECT d.file_id,
                d.file_name,
                d.bytes / 1024 / 1024 bytes_MB ,
                d.maxbytes / 1024 / 1024 maxbytes_MB,
                d.autoextensible,
                ( d.increment_by * t.block_size ) / 1024 / 1024 INCREMENT_BY_MB,
                d.tablespace_name,
                df.creation_time
         FROM   dba_data_files d,
                dba_tablespaces t,
                v$datafile df
         WHERE  d.tablespace_name = t.tablespace_name
                AND df.file# = d.file_id
         UNION ALL
         SELECT d.file_id,
                d.file_name,
                d.bytes / 1024 / 1024 bytes_MB,
                d.maxbytes / 1024 / 1024 maxbytes_MB,
                d.autoextensible,
                ( d.increment_by * t.block_size ) / 1024 / 1024 INCREMENT_BY_MB,
                d.tablespace_name,
                tf.creation_time
         FROM   dba_temp_files d,
                dba_tablespaces t,
                v$tempfile tf
         WHERE  d.tablespace_name = t.tablespace_name
                AND tf.file# = d.file_id),
free_size_mb as (select file_id,sum(bytes/1024/1024) free_MB from dba_free_space group by file_id)
SELECT allfiles.FILE_ID, allfiles.FILE_NAME, allfiles.bytes_MB, allfiles.maxbytes_MB, allfiles.autoextensible, allfiles.INCREMENT_BY_MB, allfiles.TABLESPACE_NAME, allfiles.creation_time, free_size_mb.free_MB
FROM   allfiles left join free_size_mb
on allfiles.file_id=free_size_mb.file_id
WHERE  allfiles.tablespace_name = '&1'
ORDER  BY file_id;  
```


> Context and memory play powerful roles in all the truly great meals in one's life.


<details>
<summary>My top languages</summary>

| Rank | Languages |
|-----:|-----------|
|     1| JavaScript|
|     2| Python    |
|     3| SQL       |

</details>
