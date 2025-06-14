---
layout: posts
title: OpenwebUI数据库锁处理以及数据库迁移
date: 2025-5-9 10:35:25
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "PostgreSQL"
- "sqlite"
- "锁"
- "迁移"

---



# 问题： 

Open WebUI 容器在访问 `/app/backend/data/*.sqlite` 时出现

```sh
sqlite3.OperationalError: database is locked
```

因为 SQLite 在高并发或持久卷（如 NFS/SMB）环境下容易被锁住，导致后续请求一直卡住无法读写数据库。

# 临时处理：

进入 open webUI 容器，然后 找出这个 SQLite 是哪个进程在用，直接给它杀死；

```sh
# 进入容器
docker exec -it <CONTAINER_ID> /bin/bash
# 然后在容器内运行
lsof /app/backend/data/*.sqlite
# 或者
fuser /app/backend/data/*.sqlite
# 或者
ps aux | grep sqlite 

# kill 杀死进程。
```

在宿主机重启 容器（**注意：停下需要等待长一点的时间再重启**）

```sh
docker compose down

docker compose up -d 
```



# 长期处理 （sqlite迁移PostgreSQL）：

参考：
https://linux.do/t/topic/251676/39

## 一、Docker 安装 PostgreSQL 并启动

这里为了模块化的思想贯彻到底，全部使用 docker 部署，推荐使用 docker compose。

```sh
cd /home/cys/data/docker-data
mkdir -p postgres_data
chmod 777 ./postgres_data/

cd /home/cys/docker_data
mkdir postgsql
vim docker-compose.yaml
------------------------------ docker-compose.yaml ------------------------------
services:
  postgres:
    container_name: my-postgres-sql
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: jiusdfwmij18273ee
      # 可选配置：
      # POSTGRES_USER: myuser
      # POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - /home/cys/data/docker-data/postgres_data:/var/lib/postgresql/data
    networks:
      - my-network
networks:
  my-network:
    external: true
    name: vllm-network

------------------------------ docker-compose.yaml ------------------------------

docker compose up -d 

chmod 777 /home/cys/data/docker-data/postgres_data/

docker ps # 可以看见 my_postgres_sql 开始运行了。
```



## 二、使用脚本迁移

通过 pgloader 工具 来实现 sqlite 到 PostgreSQL 的迁移，实测不能保证完全正确，无法继承全部数据。
所以我们通过 python 脚本来实现open webUI全部数据从 sqlite 到 PostgreSQL 的迁移。

### （0）准备工作：

为防迁移出错，建议先备份：

```sh
sudo cp /home/cys/data/docker-data/open_webui_data/webui.db \
   /home/cys/data/docker-data/open_webui_data/webui.db.bak
```

**生成一个空的 PostgreSQL 数据库**：
PostgreSQL 容器生成的时候就会默认生成一个 **空的 PostgreSQL 数据库**，名字叫 postgres ，我们为了图省事，直接就用这个数据库了。

**确保 PostgreSQL 已经启动**。

**注意**：记住 前面的 yaml 中的 `POSTGRES_PASSWORD:` ,是第一次生成的时候确定的，之后即便是修改了再重启容器，数据库的密码也不会变，如果要修改这个密码，需要进入容器，然后修改，再重启容器。

```sh
docker exec -it my-postgres-sql psql -U postgres

ALTER USER postgres WITH PASSWORD 'jiusdfwmij18273ee';

exit;
```

```sh
docker restart my_postgres_sql
```

 PostgreSQL 中通过 `\l` 命令查看数据库列表，确认 `postgres` 数据库存在，\dt看有哪些表。

```sh
(dockerTest) cys@cysserver:~/docker_data/postgsql$ docker exec -it my_postgres_sql psql -U postgres
psql (15.12 (Debian 15.12-1.pgdg120+1))
Type "help" for help.

postgres=# \l
```

![微信图片_20250509101133_160](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509101133_160.png)

先删除掉可能有的数据库中的表，空的就不管了。

```sql
\c postgres  -- 或 \c XXXX（如果存在其他数据库）切换到其他数据库

-- 终止所有连接到 postgres 数据库的会话
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'postgres' AND pid <> pg_backend_pid();

-- 删除目标数据库 postgres 中的表
```

看看宿主机中有没有以前 的 挂载的 文件夹也删除掉，不然可能出现 容器内数据库不匹配问题。

### （1）**关闭 Open WebUI**。

```sh
cd /home/cys/docker_data/openwebui2

docker compose down # 关闭这个 yaml 文件创建的 容器。
```

### （2）配置 Open WebUI 添加 PostgreSQL 环境变量

在open webUI 的 yaml中的 环境变量中写上：

```sh
- DATABASE_URL=postgresql://postgres:jiusdfwmij18273ee@postgres:5432/postgres
```

`postgres` 是前面的服务名。

open webUI的全部yaml文件

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:cuda
    environment:
      # 禁用 ollama 连接（注释或删除该行）
      # - OLLAMA_API_BASE_URL=http://10.5.9.252:11434
      
      # 启用 OpenAI 兼容 API（必须开启）
      - ENABLE_OPENAI_API=true
      
      # 指向本地 vLLM 的 OpenAI 兼容接口,即使你设置 container_name 为 deepseek-container，Docker内部仍然会将这个容器注册为服务名 vllm-openai，所以其他同网络的容器可以通过 “vllm-openai” 访问它
      - OPENAI_API_BASE_URL=http://vllm-openai:8000/v1
      
      # 设置与 vLLM 一致的 API 密钥
      - OPENAI_API_KEYS=jisudf*&QW123
      
      # 其他原有配置保持不变
      - GLOBAL_LOG_LEVEL=DEBUG
      - HF_ENDPOINT=https://hf-mirror.com
      - CORS_ALLOW_ORIGIN=*
      # - RAG_EMBEDDING_MODEL=bge-m3
      - DEFAULT_MODELS=deepseek-r1-70b-AWQ  # 需与 vLLM 的 --served-model-name 参数一致
      - ENABLE_OAUTH_SIGNUP=true
      # 设置默认的 Embedding 模型名称（用于内部记录或日志）
      - RAG_EMBEDDING_MODEL=ritrieve_zh_v1     # bge-large-zh-v1.5
      # 指定 Embedding API 服务的地址，确保该地址在同一网络内可访问
      - EMBEDDING_API_BASE_URL=http://10.5.9.252:8002/v1
      - EMBEDDING_API_KEYS=wiekdoid@JIDj124123     # 这里用相同的 Key 关联上
      - RAG_FILE_MAX_SIZE=10485760  # 10MB RAG 文件大小给一个最大上限。
      - DATABASE_URL=postgresql://postgres:jiusdfwmij18273ee@postgres:5432/postgres
    ports:
      - 8080:8080
    volumes:
      - /home/cys/data/docker-data/open_webui_data:/app/backend/data
    networks:
      - my-network
networks:
  my-network:
    external: true
    name: vllm-network
```

### （3）**启动 Open WebUI** 并尝试访问网站。

这一步会强制创建数据库的图表，没有这一步，迁移会失败。

```sh
cd /home/cys/docker_data/openwebui2

docker compose up -d # 或者 docker restart openwebuiXXX ...
```

### （4）**再次关闭 Open WebUI**。

```sh
cd /home/cys/docker_data/openwebui2

docker compose down 
```

### （5）**在某个位置创建一个新目录**。

```sh
mkdir /home/cys/data/docker-data/sqlite-to-postgres-migration

cd /home/cys/data/docker-data/sqlite-to-postgres-migration
```

### （6）新建一个python的迁移脚本

vim  migrate.py
主要需要填写的 是用来sqlite的位置： SQLITE_DB_PATH；以及 PostgreSQL 连接信息；这两个地方。

```python
import sqlite3
import psycopg2
import os

# Configuration
SQLITE_DB_PATH = '/home/cys/data/docker-data/open_webui_data/webui.db'  # 改成webui.db所在目录，默认为当前目录

# PostgreSQL 连接信息
PG_CONFIG = {
    'host': '10.5.9.252',
    'port': 5432,
    'database': 'postgres',
    'user': 'postgres',
    'password': 'jiusdfwmij18273ee'
}

# Helper function to convert SQLite types to PostgreSQL types
def sqlite_to_pg_type(sqlite_type):
    sqlite_type = sqlite_type.upper()
    if 'INTEGER' in sqlite_type:
        return 'INTEGER'
    elif 'REAL' in sqlite_type:
        return 'DOUBLE PRECISION'
    elif 'TEXT' in sqlite_type:
        return 'TEXT'
    elif 'BLOB' in sqlite_type:
        return 'BYTEA'
    else:
        return 'TEXT'

# Helper function to handle reserved keywords
def get_safe_identifier(identifier):
    reserved_keywords = ['user', 'group', 'order', 'table', 'select', 'where', 'from', 'index', 'constraint']
    if identifier.lower() in reserved_keywords:
        return f'"{identifier}"'
    return identifier

def migrate():
    # Connect to SQLite database
    sqlite_conn = sqlite3.connect(SQLITE_DB_PATH)
    sqlite_cursor = sqlite_conn.cursor()

    # Connect to PostgreSQL database
    pg_conn = psycopg2.connect(
        host=PG_CONFIG['host'],
        port=PG_CONFIG['port'],
        database=PG_CONFIG['database'],
        user=PG_CONFIG['user'],
        password=PG_CONFIG['password']
    )
    pg_cursor = pg_conn.cursor()

    try:
        # Get list of tables from SQLite
        sqlite_cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
        tables = sqlite_cursor.fetchall()

        for table in tables:
            table_name = table[0]

            # Skip "migratehistory" and "alembic_version" tables
            if table_name in ["migratehistory", "alembic_version"]:
                print(f"Skipping table: {table_name}")
                continue

            safe_table_name = get_safe_identifier(table_name)
            print(f"Checking table: {table_name}")

            # Check if table exists in PostgreSQL and has any rows
            try:
                pg_cursor.execute(f"SELECT COUNT(*) FROM {safe_table_name}")
                row_count = int(pg_cursor.fetchone()[0])
                
                if row_count > 0:
                    print(f"Skipping table: {table_name} because it has {row_count} existing rows")
                    continue
            except psycopg2.Error:
                # Table might not exist
                pass

            print(f"Migrating table: {table_name}")

            # Get table schema from SQLite
            sqlite_cursor.execute(f'PRAGMA table_info("{table_name}")')
            schema = sqlite_cursor.fetchall()

            # Get column names and types
            columns = []
            for col in schema:
                col_name = get_safe_identifier(col[1])
                col_type = sqlite_to_pg_type(col[2])
                columns.append(f"{col_name} {col_type}")

            # Create table in PostgreSQL if it doesn't exist
            create_table_sql = f'CREATE TABLE IF NOT EXISTS {safe_table_name} ({", ".join(columns)})'
            pg_cursor.execute(create_table_sql)
            pg_conn.commit()

            # Get table schema from PostgreSQL to determine column types
            pg_cursor.execute(
                """
                SELECT column_name, data_type
                FROM information_schema.columns
                WHERE table_name = %s
                """, 
                (table_name,)
            )
            pg_schema = pg_cursor.fetchall()
            pg_column_types = {col[0]: col[1] for col in pg_schema}

            # Get data from SQLite
            sqlite_cursor.execute(f'SELECT * FROM "{table_name}"')
            rows = sqlite_cursor.fetchall()
            
            # Get column names
            column_names = [col[1] for col in schema]
            safe_column_names = [get_safe_identifier(col) for col in column_names]
            
            # Insert data into PostgreSQL
            for row in rows:
                values = []
                for i, value in enumerate(row):
                    col_name = column_names[i]
                    if value is None:
                        values.append("NULL")
                    elif col_name in pg_column_types and pg_column_types[col_name] == 'boolean':
                        values.append('true' if value == 1 else 'false')
                    elif isinstance(value, str):
                        escaped_value = value.replace("'", "''")
                        values.append(f"'{escaped_value}'")
                    else:
                        values.append(str(value))
                
                insert_sql = f"INSERT INTO {safe_table_name} ({', '.join(safe_column_names)}) VALUES ({', '.join(values)})"
                pg_cursor.execute(insert_sql)
            
            pg_conn.commit()
            print(f"Migrated {len(rows)} rows from {table_name}")

        print("Migration completed successfully!")
    except Exception as e:
        print(f"Error during migration: {e}")
    finally:
        # Close database connections
        sqlite_conn.close()
        pg_conn.close()

if __name__ == "__main__":
    migrate()
```

### （7）运行脚本迁移

```sh
conda activate dockerTest

python migrate.py # 缺啥包补啥包即可
```

![微信图片_20250509101126_159](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509101126_159.png)

这样就是 迁移成功了。

我们还能进入 PostgreSQL 容器中 进入 数据库 postgres ，看看里面有什么表了。

```sh
docker exec -it my-postgres-sql psql -U postgres
```

```sql
\dt
```

![微信图片_20250509101137_161](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509101137_161.png)

看看user 表和 auth 检查一下，以这两个open webUI的表为 例子

```sql
SELECT * FROM auth LIMIT 10;
```

![微信图片_20250509102743_163](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509102743_163.png)

# Open WebUI 使用 PostgreSQL

重启一下 open webUI 容器

```yaml
docker compose down 
docker compose up -d
```

![微信图片_20250509102746_164](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509102746_164.png)

![微信图片_20250509102750_168](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250509102750_168.png)







# 感谢：

迁移 参考了很多  https://linux.do/t/topic/251676/19 之中的内容，感谢 **[fl0w1nd](https://linux.do/u/fl0w1nd)**![tieba_003](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/27a55a1370c41f0736ba094bdc8866c6c2878c16.png)这位大佬提供的无私教程。

![截屏2025-05-09 10.31.06](open%20webUI%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%81%E5%A4%84%E7%90%86%E4%BB%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB/%E6%88%AA%E5%B1%8F2025-05-09%2010.31.06.jpg)

