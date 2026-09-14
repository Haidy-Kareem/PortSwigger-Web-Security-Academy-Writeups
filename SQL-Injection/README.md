# SQL Injection

This folder contains write-ups for SQL Injection labs from PortSwigger 
Web Security Academy, covering in-band, blind, and error-based techniques.

## Covered Concepts
- WHERE clause manipulation
- Login bypass
- Database enumeration (Oracle / MySQL)
- UNION-based attacks
- Blind SQL Injection (conditional responses, conditional errors, time delays)
- Visible error-based SQL Injection
- Filter bypass via XML encoding

## Write-ups
- [Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](sql-injection-where-clause-hidden-data.md)
- [Lab: SQL injection vulnerability allowing login bypass](sql-injection-login-bypass.md)
- [Lab: SQL injection attack, querying the database type and version on Oracle](sql-injection-db-type-version-oracle.md)
- [Lab: SQL injection attack, querying the database type and version on MySQL](sql-injection-db-type-version-mysql.md)
- [Lab: SQL injection attack, listing the database contents on non-Oracle databases](sql-injection-db-contents-non-oracle.md)
- [Lab: SQL injection attack, listing the database contents on Oracle](sql-injection-db-contents-oracle.md)
- [Lab: SQL injection UNION attack, determining the number of columns](sql-injection-union-number-of-columns.md)
- [Lab: SQL injection UNION attack, finding a column containing text](sql-injection-union-finding-text-column.md)
- [Lab: SQL injection UNION attack, retrieving data from other tables](sql-injection-union-retrieving-data.md)
- [Lab: SQL injection UNION attack, retrieving multiple values in a single column](sql-injection-union-multiple-values.md)
- [Lab: Blind SQL injection with conditional responses](blind-sql-injection-conditional-responses.md)
- [Lab: Blind SQL injection with conditional errors](blind-sql-injection-conditional-errors.md)
- [Lab: Visible error-based SQL injection](visible-error-based-sql-injection.md)
- [Lab: Blind SQL injection with time delays](blind-sql-injection-time-delays.md)
- [Lab: Blind SQL injection with time delays and information retrieval](blind-sql-injection-time-delays-retrieval.md)
- [Lab: SQL injection with filter bypass via XML encoding](sql-injection-xml-encoding-filter-bypass.md)
