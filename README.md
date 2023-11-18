# Why Ghost on GCP
![image](https://github.com/aymanelbacha/Ghost-GCP/assets/123943459/fe923ca3-2e56-405c-a420-856d545b832e)

Do you want a beautiful content editor and a mobile-friendly control panel to host your blogs with minimum cost. instead of using Ghost managed hosting plans for a fee, while you can host Ghost (the blog engine) with its open-source and free image on GCP with few click and automaintenaned.

## GCP-Ghost Workflow Diagram
![image](https://github.com/aymanelbacha/Ghost-GCP/assets/123943459/9ea2616a-9d41-43d8-9502-5d19f2faffb9)


## Set up Google Cloud

### Pre-requisite (either)
Access through CLI console on the top right corner side
Install the gcloud CLI deoending on your environment
https://cloud.google.com/sdk/docs/install

### Initialize account
gcloud init

### Log in to the Google account
gcloud auth login

### Activate Google Cloud
gcloud auth application-default login

### Create a project
gcloud projects create ghost-blog1 --name="Ghost Blog"

### Set the project as default
gcloud config set project ghost-blog1

### Activate Free Trial
#### List the existing account then Past it next to "billing-account"
gcloud billing accounts list
gcloud alpha billing projects link ghost-blog1 --billing-account=01F1E2-A52062-2FE0EF

### Enable Compute Engine API
gcloud services enable compute.googleapis.com

### Create Instance Template (2 vCPU (1 shared core) Memory 4 GB, Disk 10 GB)
gcloud compute instance-templates create free-web-server-test1  --machine-type=e2-medium --boot-disk-size=10GB --boot-disk-type=pd-standard --boot-disk-device-name=free-web-server --tags=http-server,https-server --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any


### Create VM Instance
gcloud compute instances create ghost-blog-instance --source-instance-template=free-web-server-test1 --reservation us-central1 --zone us-central1-a


### Create Snapshot Schedule
gcloud compute resource-policies create snapshot-schedule schedule-1 --project=ghost-blog1 --region=us-central1 --max-retention-days=14 --on-source-disk-delete=apply-retention-policy --daily-schedule  --start-time=03:00 --storage-location=us --guest-flush


## Configure the domain
### Buy a domain and set up Cloudflare
1. Make a Namecheap account and buy your domain. I’ll use "aymanelbacha.pro" as an example.
2. Make a Cloudflare account and add your domain under the Free subscription plan.
3. Under “Review your DNS records”, first delete all the records.
4. Then Add record:
Type: A
Name: https://www.aymanelbacha.pro/
IPv4 address: your external IP address
Proxy status: DNS only
Save > Continue > Confirm.
5. Go back to Namecheap > aymanelbacha.pro > Manage > Nameservers > Custom DNS and input the specified Cloudflare nameservers.
6. Go back to Cloudflare > Done, check nameservers.


## Deploy Ghost
### Set up VM instance
1. Go back to Google Cloud > Compute Engine > VM Instances > ghost-blog > SSH. A virtual terminal (“cloud shell”) will appear in a pop-up window or access it through your desktop SDK via gcloud compute ssh ghost-blog-instance

2. First, set a password for the root user:
sudo passwd

3. Switch to root user and authenticate:
su

4.Update Linux:

apt update && apt -y upgrade

5.To allow any updated services to restart, go back to Google Cloud, Stop and Resume the instance, then SSH again.

6. Make a new user called service_account and grant it sudo:

adduser service_account && usermod -aG sudo service_account
Set a password for service_account. Leave all user information fields for service_account at default values. Confirm with Y.

7. Switch to service_account:

su - service_account

### Install Ghost dependencies
1. Install Nginx and open the firewall:
sudo apt install -y nginx && sudo ufw allow 'Nginx Full'

2. Install NodeJS:

sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=18
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install nodejs -y
sudo npm install -g npm@latest

3. Install MySQL:
sudo apt install -y mysql-server

Login to MySQL first:

sudo mysql
Set root password:

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password by 'MyPassword@123';
ℹ️
You should replace MyPassword@123 with a secure password that you create.
Exit:

exit;
Secure your Database Installation:

Use the provided command and adhere to the wizard's instructions to secure our Database instance:

sudo mysql_secure_installation
The script will ask these questions.

By typing your chosen password and pressing Enter, you can enter the user root password.
alter the root password? Enter after pressing N.

Take away the anonymous users? Hit Y, then hit ENTER.

Disallow remote root logins? Hit Y, then hit ENTER.

Take away access to the test database and it? Hit Y, then hit ENTER.

Now reload the privilege tables? Hit Y, then hit ENTER.

4. Create a Database for Ghost CMS
   
Use the root user's password you created for your database server to log in.

sudo mysql -u root -p

To create a new database, execute the command. But don't forget to change new_user to whatever name you want to give your database user, and new_db to whatever name you want to give your database, along with your_password for the password.

CREATE USER 'new_user'@'localhost' IDENTIFIED BY 'your_password';
CREATE DATABASE new_db;
GRANT ALL PRIVILEGES ON new_db.* TO 'new_user'@'localhost';
FLUSH PRIVILEGES;
Exit;


6. Install Ghost CLI on Ubuntu 22.04
   
Since Node.js and its package manager NPM are already installed, we can quickly install the Ghost CMS on our Ubuntu 22.04 LTS server.

sudo npm install ghost-cli@latest -g

7. Create a directory for Ghost files

sudo mkdir /var/www/ghost
sudo chown service_account:service_account /var/www/ghost
sudo chmod 775 /var/www/ghost

8. Use the CLI tool to install Ghost CMS.
   
cd /var/www/ghost && ghost install

Some questions will be posed to you by the aforementioned command:

Enter your blog URL: https://www.aymanelbacha.pro/

Enter your MySQL hostname: localhost

Enter your MySQL username: new_user

Enter your MySQL password: [hidden]

Enter your Ghost database name: new_db

Do you wish to set up Nginx? Yes

Do you wish to set up Systemd? Yes

Do you want to start Ghost? (Y/n) Y

# You will obtain the URL to access the Ghost Interface once the installation is finished.
You can run "ghost ls" to check the status of the App
