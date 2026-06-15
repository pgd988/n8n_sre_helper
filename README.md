# n8n SRE Helper Agent

This project contains an autonomous n8n workflow designed to act as an intelligent Site Reliability Engineering (SRE) assistant. It accepts alerts via Slack, automatically parses them, looks up related Runbooks in GitHub, queries Prometheus and Graylog for telemetry, and executes safe, read-only/restart infrastructure commands to diagnose or fix minor issues. Major issues are escalated to a human.

## 📦 Files Included
- **`my_n8n_workflow_template.json`**: The main Agent Orchestrator workflow.
- **`ssh_execution_subworkflow.json`**: A secure sub-workflow that enforces strict infrastructure access rules (regex validation) before executing SSH commands.

---

## 🚀 1. Importing the Workflows

Due to the security boundaries of this architecture, the infrastructure execution is decoupled from the main AI agent. You must import both workflows into your n8n UI.

### Step 1: Import the SSH Sub-Workflow
1. In your n8n UI, navigate to **Workflows** and click **Add Workflow**.
2. Click the menu (three dots) in the top right corner and select **Import from File...**
3. Select `ssh_execution_subworkflow.json`.
4. **Attach Credentials**: Double-click the `SSH` node and attach your SSH credentials (e.g., Private Key or Password) using the n8n Credential Manager.
5. Click **Save** in the top right corner.
6. Look at the URL in your browser. It will look like `https://n8n.yourdomain.com/workflow/abcdef123456`. Copy the ID part at the end (e.g., `abcdef123456`). You will need this.

### Step 2: Import the Main Agent Workflow
1. Create another new workflow and click **Import from File...**
2. Select `my_n8n_workflow_template.json`.
3. Locate the four green SSH Tool nodes attached to the AI Agent (`Docker Compose SSH`, `NGINX SSH`, `Kubernetes Kubectl`, `Kubernetes Kubectl Delete`).
4. Double-click each one of them. Find the **Workflow ID** field and paste the ID you copied from the Sub-Workflow in Step 1.
5. Save the workflow.

---

## ⚙️ 2. Configuring Global Variables

This workflow is securely configured using **n8n Global Variables** instead of `.env` files. This means you do not need to modify your Docker configuration or pass files to your container.

### How to set them up:
1. In the left-hand navigation menu of the n8n UI, click on **Settings**.
2. Go to the **Variables** tab.
3. Click **Add Variable** for each of the following keys.

### Required Variables list:

| Variable Name | Description | Example Value |
| --- | --- | --- |
| `OLLAMA_MODEL` | The LLM model you want Ollama to run. | `llama3` |
| `OLLAMA_BASE_URL` | The HTTP URL where your Ollama server is hosted. | `http://host.docker.internal:11434` |
| `PROMETHEUS_URL` | The root URL for your Prometheus instance. | `http://prometheus:9090` |
| `GRAYLOG_URL` | The root API URL for your Graylog instance. | `https://graylog.example.com/api` |
| `GRAYLOG_API_TOKEN` | Your Graylog personal access token for REST API access. | `token_1234abcd` |

| `GITHUB_RUNBOOK_REPO` | The GitHub repository in owner/repo format containing your runbooks. | `my-org/sre-runbooks` |
| `GITHUB_TOKEN` | A GitHub Personal Access Token to access the repository. | `ghp_...` |
| `SLACK_ESCALATION_WEBHOOK_URL` | The Slack incoming webhook URL used for escalating critical, unfixed issues to humans. | `https://hooks.slack.com/services/...` |
| `SLACK_REPORT_CHANNEL` | The dedicated Slack channel where the agent will post its final summary report after every iteration. | `#sre-alerts-log` |

Once these variables are saved in the UI, the workflow's `{{$vars.VARIABLE_NAME}}` expressions will automatically pull them in at execution time!

---

## 🔐 3. Configuring `kubectl` on the SSH Jump Host

Because the SRE Agent executes infrastructure commands via SSH, your n8n container does not need direct access or credentials to your Kubernetes cluster. Instead, the SSH Jump Host where the sub-workflow connects to must have `kubectl` installed and configured.

### Setup Instructions

1. **Install `kubectl`**: Ensure the `kubectl` binary is installed on the SSH jump host.
2. **Configure Kubeconfig**: The SSH user that n8n logs in as must have a valid `~/.kube/config` file configured to access your remote Kubernetes cluster.
   - If using cloud providers (AWS EKS, GCP GKE, Azure AKS), use their respective CLI tools (e.g., `aws eks update-kubeconfig`, `gcloud container clusters get-credentials`) to generate the `kubeconfig` for the SSH user.
   - Alternatively, you can manually copy a valid `kubeconfig` file to `~/.kube/config` on the SSH host.
3. **Set Context**: Ensure the default context in the `kubeconfig` is set to the cluster you want the AI agent to manage.
4. **Permissions**: Ensure the credentials used in the `kubeconfig` have the necessary RBAC permissions on the cluster (e.g., viewing pods/logs and deleting pods). You can use the provided manifests in the `kubernetes/` directory to create a highly restricted ServiceAccount (`n8n-ro-sa`), then generate a token for it and embed it into the SSH host's `kubeconfig` as the authentication mechanism.
