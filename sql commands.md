<details>
<summary>My top languages</summary>
  
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


> Context and memory play powerful roles in all the truly great meals in one's life.


<details open>
<summary>My top languages</summary>

| Rank | Languages |
|-----:|-----------|
|     1| JavaScript|
|     2| Python    |
|     3| SQL       |

</details>
