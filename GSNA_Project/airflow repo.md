
### ELT Speed Optimization and elasticsearch load balancing

Maintain and optimize the etl pipeline. Shorten the time by optimizing the dataproc resource and airflow config after adding new business data.
Guarantee pipeline finish on time and manage resources

Con

### responsible for upgrading from airflow2 to airflow3

1 debug refreshing airflow:
After airflow upgrading, some dags frequently re-load and the scheduler constantly refreshing the dag version, causing metadata DB writing pressure increase, memory allocation increase rapidly.

locate that the datetime.now() is placed in during the dag analysis, and join in the path spelling.
airflow 3 is more sensitive to the dag structure change, and recognized as dag definition change. triggering constant refresh. 
After comparing the dag logs, locate that the analysis dynamic change is the source.

Repair: Use the jinja template as substitute, like({{ ts.nodash ]}}) to join the path string

competence: from the resource altert to dag difference, and locate to the code, use smallest change for stable running


