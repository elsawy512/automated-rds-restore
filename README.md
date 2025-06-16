# Jenkins RDS Clone & Restore Pipeline

Automated Jenkins pipeline to snapshot, restore, and refresh AWS RDS PostgreSQL instances.

## 🔍 Use Case

- Clone RDS for staging/testing
- Validate disaster recovery plans
- Rebuild environments from production snapshots

## 🧰 Tools

- Jenkins
- AWS CLI
- Shell scripts
- RDS PostgreSQL

## 🔐 Security

- Uses Jenkins Credentials for secrets
- No secrets committed to code

## 🚀 Pipeline Flow

1. Rename existing DB instance
2. Create snapshot
3. Wait for snapshot
4. Restore snapshot to new DB
5. Run password and SQL scripts
6. Clean up snapshot and old DB
7. Send email notification

## ⚙️ Parameters

| Parameter        | Description                             |
|------------------|-----------------------------------------|
| `OLD_DB_NAME`    | RDS DB to rename before snapshot        |
| `RDS_INSTANCE_ID`| ID of DB to snapshot                    |
| `newInstanceId`  | New DB instance name after restore      |
| `RDS_INSTANCE_CLASS` | Instance class for new DB          |

## 📄 Scripts Required

Place the following in Jenkins script directory:

- `/var/lib/jenkins/script/CurrentDB-scripts/CurrentDB-password.sh`
- `/var/lib/jenkins/script/CurrentDB-scripts/CurrentDB-sql-scripts.sh`

Ensure they:
- Connect securely to RDS
- Run required password resets and SQL updates

## 🔔 Notifications

Job completion email sent using `emailext`.

## 📦 Setup Instructions

1. Add this repo to your GitHub organization
2. Configure Jenkins job with this repo
3. Add AWS credentials in Jenkins Credentials
4. Trigger pipeline manually or on schedule

## 📌 Requirements

- Jenkins with AWS CLI installed
- IAM role or credentials with RDS permissions:
  - `rds:Describe*`
  - `rds:CreateDBSnapshot`
  - `rds:RestoreDBInstanceFromDBSnapshot`
  - `rds:DeleteDBInstance`
  - `rds:DeleteDBSnapshot`
- PostgreSQL client (for SQL scripts)

## 🧪 Optional Enhancements

- Slack/Teams notification
- Integration with Terraform or CloudFormation
- Dynamic environment selection
- Tag-based snapshot discovery

---

## 👤 Maintainer

Engineer Ahmed  
Senior DevOps & Cloud Infra (AWS + Azure)

🎛️ Example: Parameterized Jenkinsfile
---
pipeline {
  agent any

  parameters {
    string(name: 'OLD_DB_NAME', defaultValue: 'CurrentDB', description: 'Existing RDS instance to rename')
    string(name: 'RDS_INSTANCE_ID', defaultValue: 'CurrentDB-old', description: 'RDS instance to snapshot')
    string(name: 'newInstanceId', defaultValue: 'ClonedDB', description: 'New RDS instance ID to restore to')
    string(name: 'RDS_INSTANCE_CLASS', defaultValue: 'db.m5.large', description: 'Instance type for new RDS DB')
  }

  environment {
    AWS_REGION = 'eu-west-1'
  }

  stages {
    // Same stages as before, using ${params.VARIABLE_NAME}
    // For example: ${params.OLD_DB_NAME}, ${params.newInstanceId}
  }

  post {
    always {
      emailext (
        subject: "Jenkins Job: ${currentBuild.fullDisplayName}",
        body: """
          Build result: ${currentBuild.currentResult}
          Check logs at: ${env.BUILD_URL}
        """,
        to: 'yourEmail'
      )
    }
  }
}

## 🛡 License

MIT
