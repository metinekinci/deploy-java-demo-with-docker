# Deploying a Java Demo App on an Azure VM with Docker and Nginx: A Case Study

This article provides a step-by-step guide for deploying a Java demo application on a virtual machine (VM) hosted in Azure, using Docker for containerisation and Nginx as a reverse proxy.

### Step 1: Environment Setup Using `az` Command

1. **Log in to Azure:**
   First, log in to your Azure account:
   ```bash
   az login
   ```

2. **Set your Azure subscription (if necessary):**
   If you have multiple subscriptions, set the one you want to use:
   ```bash
   az account set --subscription "Subscription_ID"
   ```

3. **Create a Resource Group:**
   Create a resource group to organize your resources:
   ```bash
   az group create --name java-demo-app --location eastus
   ```

4. **Create a Virtual Machine:**
   Use the following command to create a VM with Ubuntu 22.04 LTS:
   ```bash
   az vm create \
     --resource-group java-demo-app \
     --name demoappvm \
     --image Ubuntu2204 \
     --admin-username azureuser \
     --authentication-type ssh \
     --generate-ssh-keys \
     --size Standard_B1s
   ```

   - `--admin-username`: This is the username for the VM.
   - `--authentication-type ssh`: We use SSH for authentication.
   - `--generate-ssh-keys`: This generates an SSH key pair for accessing the VM.
   - `--size Standard_B1s`: The VM size.

5. **Open Ports for HTTP, HTTPS, and SSH:**
   Allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) by running the following commands:
   ```bash
   az vm open-port --resource-group java-demo-app --name demoappvm --port 80 --priority 1100
   az vm open-port --resource-group java-demo-app --name demoappvm --port 443 --priority 1200
   ```

6. **SSH into the VM:**
   Now, you can SSH into the VM using the public IP address:
   ```bash
   ssh azureuser@<VM_Public_IP> 
   ```
   
7. **Install Docker on the VM:**
   - Update the package database and install Docker:
     ```bash
     sudo apt update
     sudo apt install docker.io -y
     sudo systemctl start docker
     sudo systemctl enable docker
     ```
   - Verify Docker is running:
     ```bash
     docker --version
     ```

### Step 2: Run the Java Demo App

1. **Pull and run the container from the GitHub Container Registry:**
   Run the following command to pull and run the container:
   ```bash
   sudo docker run --rm -d -p 8080:8080 ghcr.io/benc-uk/java-demoapp:latest
   ```

2. **Test the application:**
   After the container is running, test that the application is accessible by running:
   ```bash
   curl http://localhost:8080
   ```

### Step 3: Install and Configure Nginx

1. **Install Nginx on the VM:**
   - Install Nginx using:
     ```bash
     sudo apt install nginx -y
     ```

2. **Configure Nginx to proxy requests to the Docker container:**
   - Open the Nginx configuration file:
     ```bash
     sudo vi /etc/nginx/sites-available/default
     ```
   
   - Replace the content of the file with the following configuration:
     ```nginx
     server {
         listen 80;
         server_name localhost;

         location / {
             proxy_pass http://localhost:8080;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     ```

   - Save the file and restart Nginx:
     ```bash
     sudo systemctl restart nginx
     ```

### Step 4: Enable HTTPS with a Self-Signed SSL Certificate

1. **Generate a self-signed SSL certificate:**
   - Create a directory for the SSL certificate:
     ```bash
     sudo mkdir /etc/nginx/ssl
     cd /etc/nginx/ssl
     ```

   - Generate the certificate and key:
     ```bash
     sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout selfsigned.key -out selfsigned.crt
     ```

   - When prompted, enter information (e.g., country, organization) or leave it blank.

2. **Configure Nginx for HTTPS:**
   - Open the Nginx configuration file again:
     ```bash
     sudo nano /etc/nginx/sites-available/default
     ```

   - Add an HTTPS server block:
     ```nginx
     server {
         listen 443 ssl;
         server_name localhost;

         ssl_certificate /etc/nginx/ssl/selfsigned.crt;
         ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

         location / {
             proxy_pass http://localhost:8080;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }

     server {
         listen 80;
         server_name localhost;

         location / {
             proxy_pass http://localhost:8080;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     ```

   - Save the changes and restart Nginx:
     ```bash
     sudo systemctl restart nginx
     ```

### Step 5: Test the Application

1. **Access the application over HTTP:**
   - Open a browser and navigate to `http://<VM_Public_IP>` to ensure the application is working via HTTP.

2. **Access the application over HTTPS:**
   - Navigate to `https://<VM_Public_IP>`. You’ll likely see a security warning due to the self-signed certificate; you can proceed by accepting the warning.

### Summary:
- **VM Setup**: Completed by creating an Azure VM and installing Docker.
- **Docker Container**: Built and ran the Java Demo App inside Docker.
- **Nginx Configuration**: Configured Nginx to proxy requests to the Docker container.
- **HTTPS Configuration**: Enabled HTTPS using a self-signed SSL certificate.
- **Testing**: Verified the application is accessible over both HTTP and HTTPS.

### Conclusion
The deployment was verified by accessing the application through both HTTP and HTTPS endpoints, ensuring that it functions as expected. As of the conclusion of this case, the VM remains active, and the application is accessible via the following public IP: `23.96.33.207`.
