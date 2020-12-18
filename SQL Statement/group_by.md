group by :
```
create table if not exists test_group_rollup2(year Int16,month Int8,day Int8) ENGINE=MergeTree() ORDER BY tuple();

INSERT INTO test_group_rollup2(year,month,day) VALUES(2019,1,5),(2019,1,15),(2020,1,5),(2020,1,15),(2020,10,5),(2020,10,15);

select year,month,day,count(1) from test_group_rollup2 group by year,month,day with rollup;
```