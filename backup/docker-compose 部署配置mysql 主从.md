使用 **Docker Compose** 搭建 **MySQL 主从复制**是一种快速、高效的方式。以下是详细的实现步骤，包括主从配置、启动和验证。

---

### **1. 环境准备**
需要安装以下工具：
- **Docker**：确保 Docker 已正确安装并运行。
- **Docker Compose**：确保可以执行 `docker-compose` 命令。

---

### **2. 编写 `docker-compose.yml` 文件**

以下是一个完整的 `docker-compose.yml` 示例文件，用于搭建 MySQL 主从架构。

```yaml
version: '3.8'

services:
  mysql-master:
    image: mysql:8.0.30
    container_name: mysql-master
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    ports:
      - "3306:3306"
    volumes:
      - ./master/my.cnf:/etc/mysql/my.cnf
      - master_data:/var/lib/mysql

  mysql-slave:
    image: mysql:8.0.30
    container_name: mysql-slave
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    ports:
      - "3307:3306"
    volumes:
      - ./slave/my.cnf:/etc/mysql/my.cnf
      - slave_data:/var/lib/mysql


volumes:
  master_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/mysql_data/mysql-master-data
  slave_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/mysql_data/mysql-slave-data

```

---

### **3. 配置主从参数**

#### **主库配置文件：`master/my.cnf`**
```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
```

#### **从库配置文件：`slave/my.cnf`**
```ini
[mysqld]
server-id=2
log-bin=mysql-bin
relay-log=mysql-relay-bin
binlog-format=ROW
read-only=1
```

---

### **4. 启动容器**
执行以下命令启动服务：
```bash
docker-compose up -d
```

- 主库运行在 `3306` 端口，从库运行在 `3307` 端口。

---

### **5. 配置主从复制**

#### **登录到主库**
```bash
docker exec -it mysql-master mysql -uroot -prootpassword
```

执行以下 SQL 命令，创建一个用于复制的用户并授予权限：
```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'replicatorpassword';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
```

记下 `File` 和 `Position` 字段的值（如 `mysql-bin.000001` 和 `154`），用于配置从库。

#### **登录到从库**
```bash
docker exec -it mysql-slave mysql -uroot -prootpassword
```

执行以下 SQL 命令，配置复制关系并启动：
```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-master',
  SOURCE_USER='replicator',
  SOURCE_PASSWORD='replicatorpassword',
  SOURCE_LOG_FILE='mysql-bin.000001', -- 根据主库的SHOW MASTER STATUS结果填写
  SOURCE_LOG_POS=154;                -- 根据主库的SHOW MASTER STATUS结果填写

START REPLICA;
SHOW REPLICA STATUS\G;
```

---

### **6. 验证主从复制**

1. **插入测试数据到主库**
   在主库中执行以下命令：
   ```bash
   docker exec -it mysql-master mysql -uroot -prootpassword
   ```
   ```sql
   USE testdb;
   CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(50));
   INSERT INTO test_table VALUES (1, 'Alice'), (2, 'Bob');
   ```

2. **检查从库是否同步**
   在从库中检查数据：
   ```bash
   docker exec -it mysql-slave mysql -uroot -prootpassword
   ```
   ```sql
   USE testdb;
   SELECT * FROM test_table;
   ```

   如果能看到插入的数据，说明主从复制配置成功！

---

### **7. 常见问题及解决**
1. **无法连接主库**
   - 检查主库的 `MYSQL_ROOT_PASSWORD` 和从库的复制用户配置是否一致。
   - 确保主库和从库能通过网络互相访问。

2. **复制中断**
   - 使用 `SHOW REPLICA STATUS\G;` 查看错误日志。
   - 常见错误包括权限不足、`File` 和 `Position` 不匹配。

3. **数据不一致**
   - 检查主从库 `my.cnf` 中的 `binlog-do-db` 和 `binlog-ignore-db` 配置是否正确。

4. **复制报错**
   - 可能是主从状态不一致照成了，可以重新设置主从状态，再进行尝试：
    从节点上运行：
```sql
   stop slave;
  reset master;
```

主节点上查看最新状态，记下 `File` 和 `Position` 字段的值（如 `mysql-bin.000001` 和 `154`），用于配置从库。
```sql
show master status; 
```

执行以下 SQL 命令，配置复制关系并启动：
```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-master',
  SOURCE_USER='replicator',
  SOURCE_PASSWORD='replicatorpassword',
  SOURCE_LOG_FILE='mysql-bin.000001', -- 根据主库的SHOW MASTER STATUS结果填写
  SOURCE_LOG_POS=154;                -- 根据主库的SHOW MASTER STATUS结果填写

START REPLICA;
SHOW REPLICA STATUS\G;
```


---

以上步骤完成后，你将成功使用 Docker Compose 搭建一个 MySQL 主从复制环境。