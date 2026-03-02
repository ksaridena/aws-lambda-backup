
## 🔐 Required Permissions
The GCP Project that hosts this script needs to have the following APIs enabled

| API | Purpose |
| :--- | :--- |
| `Cloud Run Admin API` | To host and run the Job itself. |
| `Cloud Build API` | To convert your main.py into a container image. |
| `Artifact Registry API` | To store the container image created by Cloud Build. |
| `Cloud Scheduler API` | To handle the "once every 3 months" timer. |
| `Cloud Resource Manager API` | To list the projects and folders in your Organization. |
| `Identity and Access Management API` | To check Service Accounts and their keys. |
| `API Keys API` | To check if API keys are restricted or unrestricted. |
| `Cloud Identity-Aware Proxy API` | To check for Identity-Aware Proxy clients. |
| `Gmail API` | To send the final audit report to your inbox. |
| `Cloud Logging API` | To record script errors and project activity logs. |


The executing Service Account requires these roles at the **Organization Level**:

| Role | Purpose |
| :--- | :--- |
| `roles/resourcemanager.organizationViewer` | To list and discover all projects in the Organization. |
| `roles/iam.securityReviewer` | To inspect IAM policies and Service Account key metadata. |
| `roles/serviceusage.apiKeysViewer` | To audit API key restriction settings across the Org. |
| `roles/iap.admin` | To audit OAuth/IAP client configurations. |

---

## 🛠️ Step 1: Create the Service Account (Console)
Perform these steps in the **IT-Automations** Project:

1. Navigate to **IAM & Admin > Service Accounts** in the GCP Console.
2. Click **+ CREATE SERVICE ACCOUNT**.
3. **Details**: Name it `gcp-auditor-sa` and click **Create and Continue**.
4. **Roles**: Skip this step (click **Continue**) as we will grant roles at the Org level next.
5. Click **Done**.
6. **Copy the Email address** of the new Service Account.

---

##  🛠️  Step 2: Enable APIs (Console)
Perform these steps in the **IT-Automations** Project:

1. Navigate to **APIs & Services > Enabled APIs & Services**.
2. Click **Enable APIs and Services**.
3. Search for APIs & Services in the search window, enable all the APIs listed in requirements section. 

---

## 🔑 Step 3: Assign Organization Roles (Console)
1. Switch the GCP Project Picker at the top of the console to your **Organization: aquent.com**.
2. Go to **IAM & Admin > IAM**.
3. Click **GRANT ACCESS**.
4. **Principals**: Paste the Service Account Email you copied in Step 1.
5. **Roles**: Add the 4 roles listed in the **Required Permissions** table above.
6. Click **Save**.

---

## 📧Step 4: Google Workspace Delegation (Admin Console)
To allow the Service Account to send emails via the Gmail API, a Workspace Super Admin must complete these steps:

1. **Find Client ID**: In the GCP Console, go to **IAM & Admin > Service Accounts**. Click your SA, expand **Advanced Settings**, and copy the **Unique ID** (numeric Client ID).
2. **Authorize**: Log into the [Google Workspace Admin Console](https://admin.google.com/).
3. Navigate to **Security > Access and data control > API controls > Manage Domain-wide Delegation**.
4. **Add New Entry**:
   * **Client ID**: Paste your SA's Unique ID.
   * **OAuth Scopes**: `https://www.googleapis.com/auth/gmail.send`
5. Click **Authorize**.

---

## ☁️ Step 5: Deploy Cloud Run Job (Cloud Shell)
Execute these commands from the **Cloud Shell** of the **IT-Automations** Project:

1. **Prepare Workspace**:
   ```bash
   mkdir identity-audit && cd identity-audit
   ```
2. **Add Files**.
   create **main.py** (the python script) and **requirements.txt** with the following text in the above folder
   ```requirements.txt
   google-cloud-resource-manager
   google-api-python-client
   google-auth
   ```
3. **Deploy** Replace [SA_EMAIL] with the details from **Step1** above.
   ```bash
   gcloud run jobs deploy gcp-identity-auditor \
    --source . \
    --command python3 \
    --args main.py \
    --tasks 1 \
    --region us-west1 \
    --service-account [SA_EMAIL] \
    --max-retries 0
   ```

---

## 🧪 Step 6: Testing (Console)
Verify the setup manually to ensure the automation works:

1. Go to **Cloud Run > Jobs > gcp-identity-auditor** in the GCP Console.
2. Click **Execute**.
3. Click the **Logs** tab. Look for "Process Finished" and verify that you have received the **GCP Identity Risk Audit** report in your inbox.

---

## ⏰Step 7: Quarterly Scheduling (Cloud Shell)
Schedule the script to run automatically at midnight on the 1st day of every quarter (Jan, Apr, July, Oct).
   ```bash
   gcloud scheduler jobs create http identity-audit-quarterly \
    --schedule="0 0 1 1,4,7,10 *" \
    --uri="[https://us-west1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/](https://us-west1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/)[PROJECT_ID]/jobs/gcp-identity-auditor:run" \
    --http-method=POST \
    --oauth-service-account-email=[SA_EMAIL] \
    --location=us-west1
   ```

