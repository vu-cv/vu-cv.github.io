---
layout: article
title: AWS RDS – Managed Database cho NodeJS production
tags: [aws, rds, postgresql, mysql, nodejs, cloud, database]
---
AWS RDS (Relational Database Service) là managed database — AWS lo toàn bộ: backup tự động, failover, patching, monitoring. Bài này hướng dẫn setup RDS PostgreSQL và kết nối từ NestJS.

## 1. RDS vs Self-managed

| Tiêu chí | AWS RDS | Self-managed EC2 |
|---------|---------|-----------------|
| Setup | Vài click | Phức tạp |
| Backup | Tự động (35 ngày) | Manual |
| Failover | Multi-AZ tự động | Manual |
| Patching | Tự động | Manual |
| Monitoring | CloudWatch tích hợp | Tự setup |
| Chi phí | Cao hơn ~30% | Thấp hơn |
| Control | Ít hơn | Toàn quyền |

**Nên dùng RDS khi**: Production, cần HA (High Availability), team nhỏ không có DBA.

## 2. Tạo RDS Instance

```bash
# Tạo RDS PostgreSQL với AWS CLI
aws rds create-db-instance \
  --db-instance-identifier shopxyz-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16.1 \
  --master-username dbadmin \
  --master-user-password "StrongPassword123!" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-name shopxyz \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name my-subnet-group \
  --backup-retention-period 7 \
  --multi-az \              # Tự động failover sang AZ khác
  --no-publicly-accessible  # Chỉ access từ VPC
  --region ap-southeast-1
```

## 3. Security Group — Chỉ cho phép app access

```bash
# Cho phép NestJS app trên port 5432
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-xxxxxxxx \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app-xxxxxxxx  # Security group của EC2/ECS app
```

**Không bao giờ** để RDS publicly accessible (`--publicly-accessible`).

## 4. Kết nối từ NestJS

```typescript
// src/database/database.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

TypeOrmModule.forRootAsync({
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get('DB_HOST'),      // RDS endpoint
    port: config.get<number>('DB_PORT', 5432),
    username: config.get('DB_USER'),
    password: config.get('DB_PASSWORD'),
    database: config.get('DB_NAME'),

    // SSL bắt buộc với RDS
    ssl: {
      rejectUnauthorized: true,
      ca: fs.readFileSync('rds-ca-2019-root.pem').toString(), // RDS CA cert
    },

    // Connection pool
    extra: {
      max: 20,           // Tối đa 20 connections
      min: 2,            // Duy trì ít nhất 2 connections
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    },

    entities: [__dirname + '/../**/*.entity{.ts,.js}'],
    migrations: [__dirname + '/migrations/*{.ts,.js}'],
    synchronize: false,  // KHÔNG dùng synchronize trong production!
    logging: config.get('NODE_ENV') === 'development',
  }),
  inject: [ConfigService],
}),
```

## 5. Download RDS CA Certificate

```bash
# Download cert để verify SSL
wget https://truststore.pki.rds.amazonaws.com/ap-southeast-1/ap-southeast-1-bundle.pem \
  -O rds-ca-bundle.pem
```

```typescript
// Dùng cert bundle
ssl: {
  rejectUnauthorized: true,
  ca: fs.readFileSync(path.join(__dirname, '../certs/rds-ca-bundle.pem')).toString(),
},
```

## 6. Connection String format

```env
# .env.production
DB_HOST=shopxyz-db.abc123xyz.ap-southeast-1.rds.amazonaws.com
DB_PORT=5432
DB_USER=dbadmin
DB_PASSWORD=StrongPassword123!
DB_NAME=shopxyz
DATABASE_URL=postgresql://dbadmin:StrongPassword123!@shopxyz-db.abc123xyz.ap-southeast-1.rds.amazonaws.com:5432/shopxyz?sslmode=require
```

## 7. Prisma với RDS

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

```typescript
// Prisma tự handle SSL khi URL có ?sslmode=require
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  log: process.env.NODE_ENV === 'development' ? ['query'] : [],
});
```

## 8. RDS Proxy — Connection Pooling

Khi Lambda hoặc nhiều containers kết nối RDS, cần RDS Proxy để tránh quá tải:

```bash
# Tạo RDS Proxy
aws rds create-db-proxy \
  --db-proxy-name shopxyz-proxy \
  --engine-family POSTGRESQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:..."}]' \
  --role-arn arn:aws:iam::xxx:role/rds-proxy-role \
  --vpc-subnet-ids subnet-xxx subnet-yyy \
  --vpc-security-group-ids sg-xxx
```

```typescript
// Thay endpoint trong .env
DB_HOST=shopxyz-proxy.proxy-abc123.ap-southeast-1.rds.amazonaws.com
```

RDS Proxy quản lý connection pool, kết nối Lambda/ECS nhiều instance mà không overwhelm DB.

## 9. Backup và Restore

```bash
# Tạo manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier shopxyz-db \
  --db-snapshot-identifier shopxyz-before-migration

# Restore từ snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier shopxyz-db-restored \
  --db-snapshot-identifier shopxyz-before-migration

# Point-in-time restore (bất kỳ thời điểm nào trong 35 ngày)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier shopxyz-db \
  --target-db-instance-identifier shopxyz-db-pitr \
  --restore-time 2026-01-20T10:30:00Z
```

## 10. Monitoring với CloudWatch

```typescript
// Các metric quan trọng cần theo dõi
const metrics = [
  'CPUUtilization',          // CPU > 80% → tăng instance
  'DatabaseConnections',     // Gần limit → bật RDS Proxy
  'FreeStorageSpace',        // < 20% → tăng storage
  'ReadLatency',             // > 10ms → cần index
  'WriteLatency',            // > 5ms → check slow queries
  'FreeableMemory',          // < 256MB → tăng RAM
];

// Tạo CloudWatch Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-CPU-High" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=shopxyz-db \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:...:db-alerts
```

## 11. Kết luận

- **Multi-AZ**: Bật cho production — tự động failover khi một AZ down
- **SSL**: Bắt buộc kết nối qua SSL — download RDS CA cert
- **Security Group**: Chỉ cho phép app security group access, không public
- **RDS Proxy**: Dùng khi có Lambda hoặc nhiều containers — tránh connection exhaustion
- **Automated backup**: 7 ngày minimum, point-in-time restore cho mọi sự cố

RDS đơn giản hóa database operations đáng kể — trade-off chi phí cao hơn đổi lấy reliability và thời gian quản trị ít hơn.
