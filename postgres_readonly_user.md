
# Create a read only user in Postgres in 3 easy steps:

### Harden your default installation:

```sql
    1. REVOKE ALL ON DATABASE <db_name> FROM PUBLIC;
    2. REVOKE ALL ON SCHEMA public FROM PUBLIC; 
```

These commands will remove the default access granted by the "PUBLIC" role. Now, you can grant access to individual roles.

### Provide (read-only) access to existing tables in a database/schema:
    
These commands assume that the tables in are in the 'public' schema.

```sql
    1. GRANT CONNECT ON DATABASE <db_name> TO <readonly_user>;
    2. GRANT USAGE ON SCHEMA public TO <readonly_user>;
    3. GRANT SELECT ON ALL TABLES IN SCHEMA public TO <readonly_user>;
    4. GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO <readonly_user>;
    5. GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO <readonly_user>;
```

### Provide default read-only access for newly created tables/sequences/functions:

```sql
    1. ALTER DEFAULT PRIVILEGES FOR USER <readonly_user> IN SCHEMA public GRANT SELECT ON TABLES TO <readonly_user>;
    2. ALTER DEFAULT PRIVILEGES FOR USER <readonly_user> IN SCHEMA public GRANT SELECT ON SEQUENCES TO <readonly_user>;
    3. ALTER DEFAULT PRIVILEGES FOR USER <readonly_user> IN SCHEMA public GRANT EXECUTE ON FUNCTIONS TO <readonly_user>;
```

#### Original reference: [Managing rights in postgresql][1]

----------


  [1]: http://wiki.postgresql.org/images/d/d1/Managing_rights_in_postgresql.pdf
