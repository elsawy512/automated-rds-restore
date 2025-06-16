# Jenkins Pipeline: RDS Clone and Replace Workflow

This pipeline automates the process of:
- Renaming an old RDS DB instance
- Creating a snapshot from another RDS instance
- Restoring that snapshot into a new DB
- Changing passwords and applying SQL scripts
- Cleaning up snapshot and old DB
- Sending notification upon job completion

## 💡 Purpose
Safely and repeatedly recreate the `fina-findb` instance from a production-like clone with full automation.

## 🔐 Parameters (Injected via Jenkins)
- `AWS_DEFAULT_REGION`: AWS region, default `eu-west-1`
- `RDS_INSTANCE_ID`: Source RDS instance for snapshot (e.g. `cbsprddb01-ro`)
- `newInstanceId`: New RDS instance name (e.g. `fina-findb`)
- `RDS_INSTANCE_CLASS`: Size of the restored instance (e.g. `db.m6i.xlarge`)
- `OLD_DB_NAME`: Current name of the target DB to be renamed (e.g. `fina-findb`)

## 🧩 Dependencies
- AWS CLI installed and configured on Jenkins agents
- Jenkins credentials with IAM access to manage RDS
- Custom password and SQL shell scripts:
  - `/var/lib/jenkins/script/finafindb-scripts/finafidb-password.sh`
  - `/var/lib/jenkins/script/finafindb-scripts/finafidb-sql-scripts.sh`

## 🔐 Secrets
Avoid hardcoding credentials. Use Jenkins **Credentials Binding** and **GitHub Secrets** for public sharing.

## 📁 Repository Structure
```
.
├── Jenkinsfile
├── README.md
└── scripts/
    ├── finafidb-password.sh
    └── finafidb-sql-scripts.sh
```
## 🧪 Suggested Enhancements
- Add rollback logic on failure
- Parameterize VPC/subnet/Security Group for staging reuse
- Export DB endpoint to Slack or GitHub PR comment
- Wrap in reusable shared library if standardized

---