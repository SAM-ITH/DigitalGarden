	 avialable table details are mention above all the coding examples 
- selecting a specific table and show all the details of that table 

`VILLAGE (villageid, name, chief) INHABITANT (personid, name, villageid, gender, job, gold, state)`

```sql 
select *from inhabitant
```

- show the result of relevant to a specific keyword in a table column 

`VILLAGE (villageid, name, chief) INHABITANT (personid, name, villageid, gender, job, gold, state)`

```sql 
select *from inhabitant where job = "butcher"
```

- using `AND` to find a result that true for two special condition 

`VILLAGE (villageid, name, chief) INHABITANT (personid, name, villageid, gender, job, gold, state)`

```sql
select *from inhabitant where state="friendly" and job="weaponsmith"
```

- using `LIKE` and `%`. 

