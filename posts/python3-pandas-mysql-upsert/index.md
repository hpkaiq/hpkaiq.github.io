# Python3 Pandas 实现mysql Upsert操作（唯一键更新）


python3 pandas 实现mysql upsert操作，基于唯一键有则更新，无则插入。

<!--more-->

```python
from urllib import parse
import pandas as pd
from sqlalchemy import create_engine

db_info = {'user': 'test',
           'password': parse.quote_plus('test'),
           'host': 'xxxxxx.mysql.rds.aliyuncs.com',
           'port': 3306,
           'database': 'test'
           }

engine = create_engine(
        'mysql+pymysql://%(user)s:%(password)s@%(host)s:%(port)d/%(database)s?charset=utf8' % db_info,
        encoding='utf-8')

def mysql_replace_into(table, conn, keys, data_iter):
    from sqlalchemy.dialects.mysql import insert
    data = [dict(zip(keys, row)) for row in data_iter]
    stmt = insert(table.table).values(data)
    update_stmt = stmt.on_duplicate_key_update(**dict(zip(stmt.inserted.keys(), stmt.inserted.values())))
    conn.execute(update_stmt)


def write_database(df, table_name):
    df.to_sql(table_name, con=engine, index=False, if_exists='append', method=mysql_replace_into)


if __name__ == '__main__':
    data = [
        {"app_id": 1, "app_secret": "11111122", "game_id": 11},
        {"app_id": 2, "app_secret": "22222233", "game_id": 22},
        {"app_id": 3, "app_secret": "33333344", "game_id": 33}
    ]

    detailDF = pd.DataFrame(data)
    write_database(detailDF, "ad_app_ext")

```


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/python3-pandas-mysql-upsert/  

