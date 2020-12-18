```
CREATE TABLE sample1 (id UInt64) Engine=MergeTree ORDER BY id;


INSERT INTO sample1 SELECT * FROM numbers(1000000)

-- 查看表数据路径
SELECT name, data_paths FROM system.tables WHERE name = 'sample1'

-- 查看表 parts 信息
SELECT name, disk_name, path FROM system.parts WHERE (table = 'sample1') AND active
```