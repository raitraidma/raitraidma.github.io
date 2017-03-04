## Dynamic extension tables in PostgreSQL
Let's say that we have main table that contains generic info and every entry in this table is connected to extension table that contains more specific information about that entry. We want to get necessary information in one query. For that we create a view that returns data from main table and extension table as JSON.

First let's create main table `object` and type table `object_type` which tells us, what extension table we need to use.
```sql
CREATE TABLE object_type (
  id VARCHAR PRIMARY KEY
, extension_table_name VARCHAR
);

CREATE TABLE object (
  id INT PRIMARY KEY
, object_type_id VARCHAR
, name VARCHAR
, FOREIGN KEY (object_type_id) REFERENCES object_type (id)
);

CREATE INDEX idx_object_object_type_id ON object (object_type_id);
```

Now let's create a trigger that changes the view when data in `object_type` changes.
```sql
CREATE OR REPLACE FUNCTION tf_create_object_view()
RETURNS trigger AS $$
DECLARE
  s_sql TEXT;
  s_table_name VARCHAR;
BEGIN
  s_sql := ' CREATE OR REPLACE VIEW object_view AS' ||
           ' SELECT object.*, (CASE ';

  FOR s_table_name IN (
    SELECT extension_table_name FROM object_type
  ) LOOP
    s_sql := s_sql || ' WHEN ' || s_table_name || '.id IS NOT NULL THEN to_json(' || s_table_name || ') ';
  END LOOP;

  s_sql := s_sql || ' END) AS _extension_properties ' ||
                    ' FROM object ';

  FOR s_table_name IN (
    SELECT extension_table_name FROM object_type
  ) LOOP
    s_sql := s_sql || ' LEFT JOIN ' || s_table_name || ' ON (object.id = ' || s_table_name || '.id) ';
  END LOOP;

  EXECUTE s_sql;
  RETURN NULL;
END
$$ LANGUAGE plpgsql
  SECURITY DEFINER
  SET search_path = public, pg_temp;

CREATE TRIGGER t_object_type
  AFTER INSERT OR UPDATE OR DELETE ON object_type
  FOR EACH STATEMENT
  EXECUTE PROCEDURE tf_create_object_view();
```

Finally let's create extension tables and add some data.
```sql
CREATE TABLE ext1 (
  id INT PRIMARY KEY
, ext1_column1 VARCHAR
, ext1_column2 INT
, FOREIGN KEY (id) REFERENCES object (id)
);

CREATE TABLE ext2 (
  id INT PRIMARY KEY
, ext2_column VARCHAR
, FOREIGN KEY (id) REFERENCES object (id)
);

INSERT INTO object_type (id, extension_table_name) VALUES ('EXT1', 'EXT1');
INSERT INTO object_type (id, extension_table_name) VALUES ('EXT2', 'EXT2');

INSERT INTO object(id, object_type_id, name) VALUES (1, 'EXT1', 'name1');
INSERT INTO object(id, object_type_id, name) VALUES (2, 'EXT2', 'name2');

INSERT INTO ext1(id, ext1_column1, ext1_column2) VALUES (1, 'col1', 11);
INSERT INTO ext2(id, ext2_column) VALUES (2, 'col2');
```

Let's test.
```sql
SELECT * FROM object_view;
```

And the result is:

| id | object_type_id | name  | _extension_properties                            |
|----|----------------|-------|--------------------------------------------------|
| 1  | EXT1           | name1 | {"id":1,"ext1_column1":"col1","ext1_column2":11} |
| 2  | EXT2           | name2 | {"id":2,"ext2_column":"col2"}                    |