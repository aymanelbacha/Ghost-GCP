# Why Ghost on GCP
![image](https://github.com/aymanelbacha/Ghost-GCP/assets/123943459/fe923ca3-2e56-405c-a420-856d545b832e)

Do you want a beautiful content editor and a mobile-friendly control panel to host your blogs with minimum cost. instead of using Ghost managed hosting plans for a fee, while you can host Ghost (the blog engine) with its open-source and free image on GCP with few click and automaintenaned.

## GCP-Ghost Workflow Diagram
![image](https://github.com/aymanelbacha/Ghost-GCP/assets/123943459/9ea2616a-9d41-43d8-9502-5d19f2faffb9)


## Set up Google Cloud

### Pre-requisite (either)
Access through CLI console on the top right corner side or
Install the gcloud CLI depending on your environment
https://cloud.google.com/sdk/docs/install
(we are using CF here instead as Cloud DNS in GCP took time to get published, we wanted to make sure our Instance is working fine thus we used CF)

### Initialize account
```gcloud
gcloud init
```

### Log in to the Google account
```gcloud
gcloud auth login
```

### Activate Google Cloud
```
gcloud auth application-default login
```

### Create a project
```
gcloud projects create ghost-blog1 --name="Ghost Blog"
```

### Set the project as default
```
gcloud config set project ghost-blog1
```

### Activate Free Trial
#### List the existing account then Past it next to "billing-account"
```
gcloud billing accounts list

gcloud alpha billing projects link ghost-blog1 --billing-account=01F1E5-A51062-A52062
```

### Enable Compute Engine API
```
gcloud services enable compute.googleapis.com
```

### Create Instance Template (2 vCPU (1 shared core) Memory 4 GB, Disk 10 GB)
```
gcloud compute instance-templates create free-web-server  --machine-type=e2-medium --boot-disk-size=10GB --boot-disk-type=pd-standard --boot-disk-device-name=free-web-server --tags=http-server,https-server --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```


### Create VM Instance
```
gcloud compute instances create ghost-blog-instance --source-instance-template=free-web-server --reservation us-central1 --zone us-central1-a
```

### Create Snapshot Schedule
```
gcloud compute resource-policies create snapshot-schedule schedule-1 --project=ghost-blog1 --region=us-central1 --max-retention-days=14 --on-source-disk-delete=apply-retention-policy --daily-schedule  --start-time=03:00 --storage-location=us --guest-flush
```

## Configure the domain
### Buy a domain and set up Cloudflare 
1. Make a Namecheap account and buy your domain. I’ll use "aymanelbacha.pro" as an example.
2. Make a Cloudflare account and add your domain under the Free subscription plan.
3. Under “Review your DNS records”, first delete all the records.
4. Then Add record:
Type: A
Name: https://www.aymanelbacha.pro/
IPv4 address: your external IP address -> (gcloud compute instances describe ghost-web-server-1 --format="value(networkInterfaces[0].accessConfigs[0].natIP))
Proxy status: DNS only
Save > Continue > Confirm.
5. Go back to Namecheap > aymanelbacha.pro > Manage > Nameservers > Custom DNS and input the specified Cloudflare nameservers.
6. Go back to Cloudflare > Done, check nameservers.


## Deploy Ghost
### Set up VM instance
1. Go back to Google Cloud > Compute Engine > VM Instances > ghost-blog > SSH. A virtual terminal (“cloud shell”) will appear in a pop-up window or access it through your desktop SDK via gcloud compute ssh ghost-blog-instance

2. First, set a password for the root user:
```bash
sudo passwd
```
3. Switch to root user and authenticate:
```bash
su
```
4.Update Linux:
```bash
apt update && apt -y upgrade
```
5.To allow any updated services to restart, go back to Google Cloud, Stop and Resume the instance, then SSH again.

6. Make a new user called service_user and grant it sudo:
```bash                           
adduser service_user && usermod -aG sudo service_user
```
Set a password for service_user. 
Leave all user information fields for service_user at default values. 
Confirm with Y.

7. Switch to service_user:
```
su - service_user
```
### Install Ghost dependencies
1. Install Nginx and open the firewall:
sudo apt install -y nginx && sudo ufw allow 'Nginx Full'

2. Install NodeJS:
```
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=18

echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update

sudo apt install nodejs -y

sudo npm install -g npm@latest
```
3. Install MySQL:
```
sudo apt install -y mysql-server
```
Login to MySQL first:
```
sudo mysql
```
Set root password:

ALTER USER ```'root'@'localhost' IDENTIFIED WITH mysql_native_password by 'MyPassword@123';```
ℹ️
You should replace MyPassword@123 with a secure password that you create.

Exit:
```
exit;
```
Secure your Database Installation:

Use the provided command and adhere to the wizard's instructions to secure our Database instance:
```
sudo mysql_secure_installation
```
The script will ask these questions.

By typing your chosen password and pressing Enter, you can enter the user root password.
alter the root password? Enter after pressing N.

Take away the anonymous users? Hit Y, then hit ENTER.

Disallow remote root logins? Hit Y, then hit ENTER.

Take away access to the test database and it? Hit Y, then hit ENTER.

Now reload the privilege tables? Hit Y, then hit ENTER.

4. Create a Database for Ghost CMS
   
Use the root user's password you created for your database server to log in.
```
sudo mysql -u root -p
```
To create a new database, execute the command. But don't forget to change new_user to whatever name you want to give your database user, and new_db to whatever name you want to give your database, along with your_password for the password.
```
CREATE USER 'new_user'@'localhost' IDENTIFIED BY 'your_password';
CREATE DATABASE new_db;
GRANT ALL PRIVILEGES ON new_db.* TO 'new_user'@'localhost';
FLUSH PRIVILEGES;
Exit;
```

6. Install Ghost CLI on Ubuntu 22.04
   
Since Node.js and its package manager NPM are already installed, we can quickly install the Ghost CMS on our Ubuntu 22.04 LTS server.
```
sudo npm install ghost-cli@latest -g
```
7. Create a directory for Ghost files
```
sudo mkdir /var/www/ghost

sudo chown service_user:service_user /var/www/ghost

sudo chmod 775 /var/www/ghost
```
8. Use the CLI tool to install Ghost CMS.
 ```  
cd /var/www/ghost && ghost install
```
Some questions will be posed to you by the aforementioned command:

Enter your blog URL: https://www.aymanelbacha.pro/

Enter your MySQL hostname: localhost

Enter your MySQL username: new_user

Enter your MySQL password: [hidden]

Enter your Ghost database name: new_db

Do you wish to set up Nginx? Yes

Do you wish to set up Systemd? Yes

Do you want to start Ghost? (Y/n) Y

## You will obtain the URL to access the Ghost Interface once the installation is finished.
You can run "ghost ls" to check the status of the App

## Maintenance

Create an update script in the home directory of service_user that runs every Wednesday midnight:
```
cd && sudo nano update-script.sh
```
Paste the following text into the update script:
```
#!/bin/bash

sudo apt update && sudo apt -y upgrade
sudo apt clean && sudo apt autoclean && sudo apt autoremove
sudo npm install -g npm@latest
cd /var/www/ghost
sudo npm install -g ghost-cli@latest
ghost backup
ghost stop
ghost update
ghost ls
```
Make it executable:
```
sudo chown service_usert:service_usert update-script.sh
sudo chmod 775 update-script.sh
```
create a cron job that runs every Sunday midnight to execute a script, follow these steps:

Create the cron job file: Open a text editor and create a new file. 
Save the file with the .cron extension, for example, my_wed_script.cron.

Specify the cron job schedule: In the cron job file, add the following line to define the schedule for the cron job:

0 0 * * sun
This line indicates that the cron job should run every Wed at midnight.

Specify the script path: Add the following line to the cron job file to specify the path of the script you want to execute:
/path/to/your/update-script.sh
Replace /path/to/your/script.sh with the actual path to your script file.

Save the cron job file: Save the cron job file to the appropriate location. 
The default location for cron jobs on Linux systems is /etc/cron.d.

Restart the cron daemon: To ensure the cron daemon is aware of the new cron job, restart it using the following command:

```
sudo service cron restart
```



## Security & Monitoring (GUI)
### Apply FW policies (hardening)


### Web Scan
It is recommended to keep scanning your web for vulenerabilities, using Web Scan service
![image](https://github.com/aymanelbacha/Ghost-GCP/assets/123943459/66a4fc8e-73a6-4d6f-ac5f-58cb032504f3)

### Monitoring and Notifications
Due to the fact RESTAPI wasn't working, we are showcasing how it's done from the portal (attached json file FYI)
 <p>The requested URL <code>/v3/projects/ghost-blog1/alertPoliciescurl</code> was not found on this server.  <ins>That’s all we know.</ins>
    
## Future Projects
1.Deploying on App Engine
2.Automate through Cloud Build/Functions ghost updates without running manual scripts 
3.Automate the monitoring/Web Scanning and send notifications about the overall status  

If you spot errors, vulnerabilities, or potential improvements, please do open a pull request !!!

