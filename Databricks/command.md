%fs
operate on file in file system, could cause inconsistency in unity catalog. 
Do not move file locations for the files registered in UC.
%sh
%pip
%run 
run code from other notebooks, directly use values and functions in current notebook
%run ./file_name
after running this. 

DB Utils

dbutils.fs
```python
dbutils.fs.ls('/')
// same as
%fs 


dbutils.fs.mkdirs('Workspace/Shared/Config')

config_string = """
dsgaegdsa
"""
dbutils.fs.put('Workspace/Shared/Config', config_string, True)
```

dbutils.widgets
```python
dbutils.widgets.text("widget_name", degault_val)
text = dbutils.widgets.get("widget_name")
```

dbutils.notebook
```python
# Define a exit code
count = 20
dbutils.notebook.exit(count)

# result will get the defined exit code for the notebook
result = dbutils.notebook.run("../.../notebook_name", timeout_second)


```