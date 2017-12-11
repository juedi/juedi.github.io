title: queryForLong
date: 2017-12-11 13:35:04
categories: java

---

使用spring jdbcTemplate查询单值结果时，例如queryForLong、queryForInt这种，如果数据库返回的结果为空，就会报错，而我们希望的结果，一般都是有值的话就返回值，没值的花就返回null
这样，就要使用这种方式

```
jdbcTemplate.query(sql, values, new ResultSetExtractor<Long>(){
    @Override
    public Long extractData(ResultSet rs) throws SQLException, DataAccessException{
        if(rs.next()){
            return rs.getLong(column);
        }
        return null;
    }
})
```

不能采用这样的方式

```
jdbcTemplate.queryForObject(sql, values, Long.class);
```

也不能采用


```
jdbcTemplate.queryForObject(sql, values, someRowMapper)
```

