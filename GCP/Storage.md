Lock policy on retention can not be reverted

CRC32 compare the differernce between files
gsutil hash -c (calculate the on-prem file hash)
gsutil -L list gs:// (calculate the gcs hash)

Data transfer:
Appliance: for local large than 10Tb
storage transfer service: 3 party. Suitable for 大批量规模传输 不适合单个大文件传输
gsutil: small

Backup:
PD snapshot: google managed storage
BQ: GCS
Cloud SQL: GCS
Backup and DR(Backup and DR service): backup 

GCS fuse:
use gcs as the mounted path as a pd

MemoCache:
shared: Use with others. Dedicated: use alone, for prod