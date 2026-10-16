```python
big = df[df['amount'] > 100]

rev = df.groupby("segment")["amount"].sum()

joined = order.merge(customer, on= "customer_id")

```

```python
#select
df[[cols]]

#where
df[mask] df.loc[mask]

#groupby + sum
df.groupby().agg()

#join
df.merge(other, on = key)

# order_by
df.sort_values()

# distinct
df.drop_duplicates()

```

```python
df[mask]['status'] = 'large'
#will not change the original df
# df[mask] will only get a copy of the df

df.loc[mask, 'status'] = 'large' # write to the df
df.loc[df['amount']>100, "status"] = 'large'

# create a new df explicitly
big = df[df['amount']>100].copy()
big['status'] = 'large'

```
- Vectorization：优先选择，适合整列计算和条件操作。
    
- `itertuples()`：确实需要逐行执行 Python 逻辑时，通常优于 `iterrows()`。
    
- `iterrows()`：使用方便，但通常更慢，并且要注意数据类型转换。
- 
方法 A：逐行处理

较慢

```python
df["bonus"] = 0

for i, row in df.iterrows():
    if row["salary"] > 5000:
        df.loc[i, "bonus"] = row["salary"] * 0.1
```


方法 B：向量化

推荐

```python
df["bonus"] = (
    df["salary"]
    .where(df["salary"] > 5000, 0)
    * 0.1
)
```

通过整列条件判断和运算，一次性计算所有行的奖金。

## loc iloc常见用法对比

|操作|`loc`|`iloc`|
|---|---|---|
|取一行|`df.loc[20]`|`df.iloc[1]`|
|取一列|`df.loc[:, "salary"]`|`df.iloc[:, 1]`|
|取一个单元格|`df.loc[20, "salary"]`|`df.iloc[1, 1]`|
|取多行|`df.loc[[10, 30]]`|`df.iloc[[0, 2]]`|
|取多列|`df.loc[:, ["name", "salary"]]`|`df.iloc[:, [0, 1]]`|
`loc` 可以直接用条件筛选

df.loc(mask, [])

修改原df
```python
df.loc[0, "salary"] = 6000
df.iloc[1, 1] = 8000
```

使用返回新结果的方法

```python
df2 = df.drop(columns=["salary"])
df3 = df.sort_values("salary")
df4 = df.reset_index(drop=True)
```

情况 C：使用返回新结果的方法

```
df2 = df.drop(columns=["salary"])df3 = df.sort_values("salary")df4 = df.reset_index(drop=True)
```

## 特别注意：链式赋值（Chained Assignment）

这是 Pandas 面试中很常见的陷阱。

```python
df[df["salary"] > 5000]["salary"] = 0
```

不要这样写来修改原始 DataFrame。

它先筛选行，再对筛选结果赋值。在 Pandas 2.x 的不同版本和 Copy-on-Write 设置下，这种操作可能无法修改原始 `df`，也可能产生警告。

正确写法：

```python
df.loc[df["salary"] > 5000, "salary"] = 0
```

Use generator to read large files
```python
def parsed(path):
	with open(path) as f:
		for line in f:
			yield parse(line)

for row in parsed("data.txt"):
    print(row)

```

in pandas
```python
for chunk in pd.read_csv("event.csv", chunksize = 10000):
	process(chunk)

```