# Google Cloud Data Engineer：Storage 知识整理

## 1. Cloud Storage 数据保护与 Retention Policy

### 1.1 Retention Policy

Cloud Storage Retention Policy 用于规定 Object 至少需要保留多长时间，在保留期限内不能被删除或覆盖。

例如：

```
Retention Period = 30 days

Object Created
      │
      ↓
  30-day retention
      │
      ├── Delete → Not allowed
      ├── Overwrite → Not allowed
      └── Read → Allowed
      │
      ↓
After 30 days
      │
      ↓
Object can be deleted or replaced
```

主要概念：

- Retention Period：对象必须保留的最短时间。
    
- Retention Policy：Bucket 级别的保留规则，作用于 Bucket 中的对象。
    
- Locked Retention Policy：锁定后的保留策略，具有更强的不可逆约束。
    
- Bucket Lock：将 Bucket 的 Retention Policy 锁定，满足 WORM（Write Once, Read Many）类型的数据保护需求。
    

### 1.2 Locked vs Unlocked Retention Policy

|比较|Unlocked|Locked|
|---|---|---|
|修改 Retention Period|可以调整|只能增加，不能减少|
|删除 Retention Policy|可以|不可以|
|是否可逆|可以修改或移除|不可解锁|
|适用场景|普通数据保留|法规、审计、合规数据保护|

考试重点

一旦锁定 Retention Policy，就不能解除锁定，也不能删除该策略或缩短保留期限。

锁定前应该确认：

- Retention Period 是否正确。
    
- 是否满足法规要求。
    
- 是否会影响正常的数据清理。
    
- 是否有足够的存储成本预算。
    

### 1.3 Retention Policy vs Lifecycle Management

这两个功能容易混淆。

|比较|Retention Policy|Lifecycle Management|
|---|---|---|
|目的|防止数据过早删除或覆盖|自动管理对象生命周期|
|典型操作|限制删除、覆盖|删除、转存储类别|
|是否强制保留|是|否|
|常见用途|合规、审计、数据保护|成本优化、自动清理|

例如：

- Retention Policy：日志必须保留 365 天。
    
- Lifecycle Rule：日志超过 365 天后删除。
    

如果 Retention Policy 仍然禁止删除，即使 Lifecycle Rule 触发了删除条件，也不能提前删除对象。

面试问题

1. What is a Cloud Storage Retention Policy?
    
2. What is the difference between a locked and unlocked retention policy?
    
3. Can you unlock a locked retention policy?
    
4. What is the difference between Retention Policy and Lifecycle Management?
    
5. How would you implement WORM storage for compliance requirements?
    

## 2. Data Integrity：CRC32C 与文件校验

### 2.1 为什么需要 Checksum？

数据在传输过程中可能出现：

- 文件损坏。
    
- 网络传输错误。
    
- 数据不完整。
    
- 源文件和目标文件内容不一致。
    

因此，需要通过 Checksum 验证源文件和目标文件是否一致。

Google Cloud Storage 使用 CRC32C 作为主要的数据完整性校验机制。

### 2.2 CRC32C

CRC32C（Cyclic Redundancy Check）是一种基于多项式计算的校验算法，用于检测数据传输或存储过程中的意外损坏。

基本流程：

```
On-premises File
       │
       ↓
Calculate CRC32C
       │
       ↓
Upload to GCS
       │
       ↓
GCS calculates CRC32C
       │
       ↓
Compare Checksums
       │
       ├── Match → Data integrity verified
       │
       └── Mismatch → Data corruption detected
```

如果源文件和 GCS 对象的 CRC32C 一致，可以认为文件内容通过了 CRC32C 完整性校验。

注意：CRC32C 主要用于检测意外损坏，不是用于防止恶意篡改的加密 Hash。

### 2.3 使用 gsutil 检查 Hash

计算本地文件 CRC32C：

```
gsutil hash -c local_file.csv
```

其中：

- `-c`：计算 CRC32C。
    
- `-m`：不是 Hash 参数，而是 gsutil 并行处理参数，不能与 `-c` 混淆。
    

查看 GCS 对象 Hash：

```
gsutil ls -L gs://my-bucket/my-file.csv
```

输出中可以查看：

```
Hash (crc32c): ...
Hash (md5): ...
```

然后比较本地文件和 GCS 对象的 CRC32C。

注意：

- CRC32C 和 MD5 是不同算法，不能直接比较两种算法的结果。
    
- GCS 对象的 MD5 在某些上传类型或对象场景下可能不存在。
    
- 使用 CRC32C 对比时，必须确保两边使用相同的算法和编码格式。
    

面试问题

1. What is CRC32C, and why is it used in Cloud Storage?
    
2. How do you verify that an uploaded file is not corrupted?
    
3. What is the difference between CRC32C and MD5?
    
4. Does CRC32C protect data against malicious modification?
    
5. How would you validate data integrity when transferring files from an on-premises environment to GCS?