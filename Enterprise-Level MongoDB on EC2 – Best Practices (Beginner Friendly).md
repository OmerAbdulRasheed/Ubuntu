# Enterprise-Level MongoDB on EC2 – Best Practices (Beginner Friendly)

---

## Introduction

MongoDB is a popular NoSQL database known for its flexibility and scalability. Running MongoDB on **AWS EC2** gives you full control over the database environment, but it also comes with operational responsibilities. This guide walks you through **enterprise-level best practices** for running MongoDB on EC2 — including setup, security, backup, and performance — in a way that's **friendly for beginners**.

---

## Step 1: Choose the Right EC2 Instance

* **Memory**: MongoDB benefits from RAM — choose an instance with enough memory to hold your working set.
* **CPU**: Ensure the CPU can handle your workload; burstable instances (like `t3`) are okay for moderate loads.
* **Storage**: Use **EBS volumes** instead of ephemeral storage for durability.
* **Example**: `t3.xlarge` or `m5.large` for medium workloads.

---

## Step 2: Prepare EBS Volumes for MongoDB Data

```bash
# Create mount directory
sudo mkdir -p /data/db

# Format EBS volume (replace /dev/nvme1n1 with your volume)
sudo mkfs -t xfs /dev/nvme1n1

# Mount the volume
sudo mount /dev/nvme1n1 /data/db

# Set permissions for MongoDB
sudo chown -R mongodb:mongodb /data/db

# Make mount persistent
echo "/dev/nvme1n1 /data/db xfs defaults,noatime 0 0" | sudo tee -a /etc/fstab
```

---

## Step 3: Install MongoDB

```bash
# Import MongoDB public GPG key
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Add MongoDB repository
echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update packages and install MongoDB
sudo apt update
sudo apt install -y mongodb-org

# Start and enable MongoDB service
sudo systemctl enable mongod
sudo systemctl start mongod
sudo systemctl status mongod
```

* Point MongoDB to your EBS-mounted data directory in `/etc/mongod.conf`:

```yaml
storage:
  dbPath: /data/db
  journal:
    enabled: true
```

---

## Step 4: Security Best Practices

```bash
# Create admin user inside mongosh
mongosh
use admin
db.createUser({ user: "admin", pwd: "StrongPasswordHere123!", roles: [{ role: "root", db: "admin" }] })
exit

# Enable authentication in /etc/mongod.conf
sudo nano /etc/mongod.conf
# add:
security:
  authorization: "enabled"

# Restart MongoDB
sudo systemctl restart mongod
```

1. Bind to localhost: `bindIp: 127.0.0.1` in `mongod.conf`.
2. Strong passwords for admin/app users.
3. VPC security groups restrict access.
4. TLS/SSL for encrypted connections (optional).

---

## Step 5: Backup and Disaster Recovery

```bash
# Logical backup
mongodump --db mydatabase --archive=/tmp/mydatabase.archive --gzip

# Restore backup
mongorestore --archive=/tmp/mydatabase.archive --gzip --db mydatabase -u admin -p StrongPasswordHere123! --authenticationDatabase admin

# Create EBS snapshot
aws ec2 create-snapshot --volume-id <volume-id> --description "MongoDB backup"
```

* Use **replica sets** for high availability across multiple AZs.

---

## Step 6: Monitoring and Maintenance

```bash
# Check MongoDB logs
tail -f /var/log/mongodb/mongod.log

# Check system resource usage
htop
iostat -x 5
free -h
```

* Use **CloudWatch** or **Ops Manager** for monitoring.
* Regularly update Ubuntu and MongoDB.

---

## Step 7: Migration from Atlas (Optional)

```bash
# Dump Atlas database
mongodump --uri "mongodb+srv://username:password@cluster0.mongodb.net/mydatabase" --archive=/tmp/mydatabase.archive --gzip

# Restore to EC2
mongorestore --archive=/tmp/mydatabase.archive --gzip --db mydatabase -u admin -p StrongPasswordHere123! --authenticationDatabase admin
```

* Verify data:

```javascript
show dbs
use mydatabase
show collections
db.collectionName.countDocuments()
```

---

## Step 8: Connect Your Application

* Local MERN app (no auth):

```env
MONGO_URI=mongodb://localhost:27017/mydatabase
```

* With authentication:

```env
MONGO_URI=mongodb://admin:StrongPasswordHere123!@localhost:27017/mydatabase?authSource=admin
```

---

## Step 9: Performance Optimization Tips

```bash
# Create indexes for frequent queries
mongosh
use mydatabase
db.users.createIndex({ email: 1 })
```

* Keep working set in RAM.
* Replica sets for read scaling.
* Disable Linux swap for better performance:

```bash
sudo sysctl vm.swappiness=1
```

---

## Step 10: Summary & Best Practices Checklist

| Area           | Best Practice                                                        |
| -------------- | -------------------------------------------------------------------- |
| Security       | Enable auth, bind to localhost, use strong passwords, secure network |
| Reliability    | Use EBS volumes, backups, replica sets                               |
| Performance    | Choose proper EC2 instance, tune memory/CPU, indexes                 |
| Monitoring     | Logs, CloudWatch, Ops Manager                                        |
| Data Migration | Use `mongodump`/`mongorestore` safely                                |

---

## Conclusion

Running MongoDB on EC2 gives you **full control and flexibility**, but it requires careful planning. By following these **enterprise-level best practices**, you can achieve **secure, reliable, and performant MongoDB deployments** that scale with your applications.

---

### Optional Next Steps

* Add **diagrams** for EC2 + EBS + Replica Sets.
* Include **automation scripts** (Terraform/Bash).
* Write a **follow-up on monitoring and alerting** for MongoDB.
