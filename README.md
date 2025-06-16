Automated RDS Clone & Restore with Jenkins CI

🔍 Use Case

Rebuild and clone an AWS RDS DB instance safely after production snapshotUsed for staging refresh, DR validation, or test DB rebuild

🧠 Problem

Manual RDS snapshot & restore is error-prone and slow

Risk of data leaks or downtime

Engineers wait hours to get updated test data

🔧 Solution

Jenkins pipeline automates:

Snapshot creation

Instance renaming

Safe snapshot restore

Custom password + SQL execution

Cleanup

🏗️ Tools

Jenkins

AWS CLI

Shell scripting

RDS PostgreSQL

🔐 Security

Secrets managed via Jenkins Credentials

No keys committed

📦 Pipeline Stages
1. Rename old DB
2. Create snapshot
3. Wait for snapshot ready
4. Restore to new instance
5. Run post-restore scripts
6. Clean up old DB + snapshot
7. Notify by email

📄 Pipeline File (Jenkinsfile)
pipeline {
  agent any

  stages {
    stage('Rename CurrentDB to Old') {
      steps {
        script {
          def oldExists = sh(
            script: "aws rds describe-db-instances --db-instance-identifier ${OLD_DB_NAME} --query 'DBInstances[0].DBInstanceIdentifier' --output text || echo notfound",
            returnStdout: true
          ).trim()

          if (oldExists == OLD_DB_NAME) {
            echo "Renaming ${OLD_DB_NAME} to CurrentDB-old..."
            sh """
              aws rds modify-db-instance \
                --db-instance-identifier ${OLD_DB_NAME} \
                --new-db-instance-identifier CurrentDB-old \
                --apply-immediately
            """
            sleep(100)

            timeout(time: 30, unit: 'MINUTES') {
              waitUntil {
                def status = sh(
                  script: "aws rds describe-db-instances --db-instance-identifier CurrentDB-old --query 'DBInstances[0].DBInstanceStatus' --output text",
                  returnStdout: true
                ).trim()
                echo "⏳ Waiting for rename... Current status: ${status}"
                return status == "available"
              }
            }
            echo "✅ Renamed successfully"
          } else {
            echo "No existing instance to rename."
          }
        }
      }
    }

    stage('Create RDS Snapshot') {
      steps {
        script {
          def now = new Date().format("yyyyMMddHHmmss")
          def snapshotId = "rds-snap-${RDS_INSTANCE_ID}-${now}"
          sh "aws rds create-db-snapshot --db-instance-identifier ${RDS_INSTANCE_ID} --db-snapshot-identifier ${snapshotId}"
          env.RDS_SNAPSHOT_ID = snapshotId
        }
      }
    }

    stage('Wait for RDS Snapshot to be Available') {
      steps {
        script {
          timeout(time: 30, unit: 'MINUTES') {
            waitUntil {
              def status = sh(
                script: "aws rds describe-db-snapshots --db-snapshot-identifier ${env.RDS_SNAPSHOT_ID} --query 'DBSnapshots[0].Status' --output text",
                returnStdout: true
              ).trim()
              echo "⏳ Waiting for RDS snapshot... Current status: ${status}"
              return status == "available"
            }
          }
          echo "✅ RDS Snapshot is available!"
        }
      }
    }

    stage('Restore RDS Snapshot to New Instance') {
      steps {
        script {
          sh """
            aws rds restore-db-instance-from-db-snapshot \
              --db-instance-identifier ${newInstanceId} \
              --db-snapshot-identifier ${env.RDS_SNAPSHOT_ID} \
              --db-instance-class ${RDS_INSTANCE_CLASS} \
              --availability-zone eu-west-1a \
              --no-multi-az \
              --db-subnet-group-name integration-prod-subnet \
              --vpc-security-group-ids sg-0155f2c34f345a106 \
              --license-model bring-your-own-license \
              --option-group-name prdoptiongroup-19-1 \
              --storage-type io1 \
              --iops 3000 \
              --enable-cloudwatch-logs-exports alert audit listener trace \
              --tags Key=Name,Value=${newInstanceId}
          """
        }
      }
    }

    stage('Wait for Restored RDS to be Available') {
      steps {
        script {
          timeout(time: 40, unit: 'MINUTES') {
            waitUntil {
              def status = sh(
                script: "aws rds describe-db-instances --db-instance-identifier ${newInstanceId} --query 'DBInstances[0].DBInstanceStatus' --output text",
                returnStdout: true
              ).trim()
              echo "⏳ Waiting for restored RDS... Current status: ${status}"
              return status == "available"
            }
          }

          def endpoint = sh(
            script: "aws rds describe-db-instances --db-instance-identifier ${newInstanceId} --query 'DBInstances[0].Endpoint.Address' --output text",
            returnStdout: true
          ).trim()

          env.RDS_ENDPOINT = endpoint
          echo "✅ Using endpoint: ${env.RDS_ENDPOINT}"
        }
      }
    }

    stage('Change Root Password & Apply Custom Scripts') {
      steps {
        sh '/var/lib/jenkins/script/CurrentDB-scripts/CurrentDB-password.sh'
        sleep(30)
        sh '/var/lib/jenkins/script/CurrentDB-scripts/CurrentDB-sql-scripts.sh'
      }
    }

    stage('Cleanup: Delete RDS Snapshot & Old DB') {
      steps {
        script {
          echo "Deleting created snapshot: ${env.RDS_SNAPSHOT_ID}..."
          sh "aws rds delete-db-snapshot --db-snapshot-identifier ${env.RDS_SNAPSHOT_ID}"

          timeout(time: 15, unit: 'MINUTES') {
            waitUntil {
              def snapshotStatus = sh(
                script: "aws rds describe-db-snapshots --db-snapshot-identifier ${env.RDS_SNAPSHOT_ID} --query 'DBSnapshots[0].Status' --output text || echo deleted",
                returnStdout: true
              ).trim()
              echo "⏳ Waiting for snapshot deletion... Current status: ${snapshotStatus}"
              return snapshotStatus == "deleted" || snapshotStatus == "notfound"
            }
          }
          echo "✅ Snapshot deleted successfully."

          echo "Deleting old database: CurrentDB-old..."
          sh "aws rds delete-db-instance --db-instance-identifier CurrentDB-old --skip-final-snapshot"

          timeout(time: 20, unit: 'MINUTES') {
            waitUntil {
              def dbStatus = sh(
                script: "aws rds describe-db-instances --db-instance-identifier CurrentDB-old --query 'DBInstances[0].DBInstanceStatus' --output text || echo deleted",
                returnStdout: true
              ).trim()
              echo "⏳ Waiting for old database deletion... Current status: ${dbStatus}"
              return dbStatus == "deleted" || dbStatus == "notfound"
            }
          }
          echo "✅ Old database deleted successfully."
        }
      }
    }
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
