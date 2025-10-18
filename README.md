# DevOps Project: Python Flask App on Azure K3s with CI/CD

This project demonstrates setting up a DevOps infrastructure on Microsoft Azure to deploy a simple Python Flask web application. It uses Infrastructure as Code (IaC), configuration management, containerization, orchestration, and CI/CD automation.

**Project Report:** For a detailed explanation, refer to the [Rapport_Projet_DevOps.pdf](Rapport_Projet_DevOps.pdf) included in this repository.

## Technologies Used

* **Cloud Provider:** Microsoft Azure
* **Infrastructure as Code (IaC):** Terraform
* **Configuration Management:** Ansible
* **Containerization:** Docker
* **Container Orchestration:** K3s (Lightweight Kubernetes)
* **CI/CD:** Jenkins
* **Version Control:** Git & GitHub
* **Application:** Python Flask Demo App

## Architecture Overview

1.  **Terraform** provisions the necessary Azure resources (Resource Group, VNet, Subnet, NSG, Public IPs, VMs for master and worker nodes).
2.  **Ansible** configures the provisioned VMs by installing Docker, K3s, kubectl, and setting up the Kubernetes cluster.
3.  **Jenkins**, running on the master node, orchestrates the CI/CD pipeline.
4.  When code is pushed to **GitHub**, a webhook triggers the **Jenkins pipeline**.
5.  The pipeline checks out the code, installs dependencies, runs tests, builds a **Docker** image, pushes it to Docker Hub, and deploys the application to the **K3s cluster**.
6.  **Kubernetes (K3s)** manages the running application container.

*(Refer to Figure 2.1 in the project report for a visual diagram)*

## Prerequisites

Before you begin, ensure you have the following installed and configured:

1.  **Azure Account:** An active Azure subscription.
2.  **Azure CLI:** Installed and logged in (`az login`).
3.  **Terraform:** [Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli).
4.  **Ansible:** [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html). (Usually runs on Linux/WSL/macOS).
5.  **Git:** [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git).
6.  **Docker Hub Account:** To store the built Docker images.
7.  **SSH Key Pair:** An SSH key pair generated (e.g., using `ssh-keygen`). The public key path needs to be correctly referenced in `Terraform/main.tf`. Ensure the private key is available for Ansible connections.
8.  **(Optional) WSL:** If using Windows, WSL (Windows Subsystem for Linux) is recommended for running Ansible and Terraform.

## Setup and Deployment Steps

1.  **Clone the Repository:**
    ```bash
    git clone <your-repository-url>
    cd Projet_DevOps
    ```

2.  **Configure Azure Credentials:**
    * Make sure you are logged into Azure CLI:
        ```bash
        az login
        ```
    * Set the subscription if you have multiple:
        ```bash
        az account set --subscription "<your-subscription-id>"
        ```

3.  **Provision Infrastructure with Terraform:**
    * Navigate to the Terraform directory:
        ```bash
        cd Terraform
        ```
    * **(IMPORTANT)** Update the `public_key` path in `main.tf` to point to *your* SSH public key file:
        ```terraform
        # Inside the azurerm_linux_virtual_machine resources...
        admin_ssh_key {
          username   = "azureuser"
          public_key = file("~/.ssh/id_rsa.pub") # <-- UPDATE THIS PATH
        }
        ```
    * Initialize Terraform:
        ```bash
        terraform init
        ```
    * Apply the Terraform configuration to create Azure resources:
        ```bash
        terraform apply -auto-approve
        ```
    * **Note down the Public IP addresses** for `masterVM` and `worker1VM` displayed in the Terraform output.

4.  **Configure VMs with Ansible:**
    * Navigate to the Ansible directory:
        ```bash
        cd ../Ansible
        ```
    * Create an Ansible inventory file named `inventory.ini` (or any name you prefer) with the public IPs obtained from Terraform:
        ```ini
        [masters]
        <masterVM_public_ip>

        [workers]
        <worker1VM_public_ip>

        [all:vars]
        ansible_user=azureuser
        ansible_ssh_private_key_file=~/.ssh/id_rsa # <-- UPDATE path to your private key if different
        ansible_python_interpreter=/usr/bin/python3
        ```
    * Run the Ansible playbook to configure the VMs:
        ```bash
        ansible-playbook -i inventory.ini playbook.yml
        ```
        *(This will install Docker, K3s, kubectl etc.)*

5.  **Set Up Jenkins:**
    * Access the Jenkins UI in your browser using the Master VM's public IP on port 8080: `http://<masterVM_public_ip>:8080`.
    * Complete the initial Jenkins setup:
        * Retrieve the initial admin password from the master VM: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
        * Install suggested plugins.
        * Create your first admin user.
    * Add necessary credentials in Jenkins (**Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials (unrestricted)** -> **Add Credentials**):
        * **Docker Hub Credentials:** Kind: `Username with password`, ID: `docker-hub-credentials`, Username: `<your-dockerhub-username>`, Password: `<your-dockerhub-password>`.
        * **Kubeconfig:** Kind: `Secret file`, File: Upload `/etc/rancher/k3s/k3s.yaml` from your master node (you might need to `scp` it first or copy its content), ID: `azure-kubeconfig`. *Note: The Jenkinsfile currently uses the default path `/var/lib/jenkins/.kube/config`, you might need to adjust the Jenkinsfile or ensure the kubeconfig is placed there via Jenkins config/startup.* The report mentions using `/etc/rancher/k3s/k3s.yaml`, ensure consistency.
        * **(Optional) GitHub Credentials:** If your repository is private or for secure webhook integration.
    * Configure the **GitHub Webhook** in your GitHub repository settings (**Settings** -> **Webhooks** -> **Add webhook**):
        * Payload URL: `http://<masterVM_public_ip>:8080/github-webhook/`.
        * Content type: `application/json`.
        * Leave Secret empty for now (or configure if needed).
        * Select "Just the push event".
        * Ensure the webhook is active and successfully delivering ping events.
    * Create the Jenkins Pipeline Job:
        * Click **New Item**.
        * Enter a name (e.g., `DevOps-Project-Pipeline`).
        * Select **Pipeline** and click OK.
        * In the configuration page, scroll down to the **Pipeline** section.
        * Definition: Select **Pipeline script from SCM**.
        * SCM: Select **Git**.
        * Repository URL: Enter your GitHub repository URL (e.g., `https://github.com/YourUsername/Projet_DevOps.git`).
        * Branch Specifier: `*/main`.
        * Script Path: `Jenkins/Jenkinsfile`.
        * Click **Save**.

6.  **Trigger the Jenkins Pipeline:**
    * Make a small change to your application code (e.g., update text in a template) and push it to the `main` branch of your GitHub repository.
    * Alternatively, manually trigger the pipeline from the Jenkins UI by clicking **Build Now** on the pipeline job page.
    * Monitor the pipeline execution in Jenkins through the "Build History" and "Console Output". The pipeline will perform checkout, dependency install, tests, Docker build/push, and Kubernetes deployment.

7.  **Access the Deployed Application:**
    * SSH into your **master** VM: `ssh azureuser@<masterVM_public_ip>`
    * Find the NodePort assigned to your service:
        ```bash
        kubectl get svc python-app-service
        ```
        Look for the port mapping under the `PORT(S)` column (e.g., `5000:32694/TCP` - `32694` is the NodePort in this example).
    * Open your web browser and navigate to: `http://<worker1VM_public_ip>:<NodePort>` (use the NodePort you found in the previous step). You should see your Python Flask application running.

## Cleanup

To avoid ongoing Azure charges, destroy the created infrastructure when you are finished:

1.  Navigate to the `Terraform` directory:
    ```bash
    cd ../Terraform
    ```
2.  Run the destroy command:
    ```bash
    terraform destroy -auto-approve
    ```

---

Feel free to modify this template, add more details from your report, or include sections like troubleshooting tips! Remember to add this `README.md` file to Git, commit it, and push it to your repository.