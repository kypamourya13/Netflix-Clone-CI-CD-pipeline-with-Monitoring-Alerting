# Netflix-Clone-CI/CD-pipeline-with-Monitoring-and-Alerting | Email

![DEVOPS](https://github.com/rutikdevops/DevOps-Project-11/assets/109506158/9fabe5eb-790f-40ac-9fc5-05f1b48cae33)

# Project Overview ::
<p align="justify"> We will be deploying a Netflix clone into a Kubernetes cluster. A Netflix clone project is a web application that replicates the functionality and features of the original Netflix platform. The project involves creating a user-friendly interface. We will be using Jenkins as a CICD tool and deploying our application on a Docker container and Kubernetes Cluster and we will monitor the Jenkins and Kubernetes metrics using Grafana, Prometheus and Node exporter. </p>


# Project Steps ::
- Step 1   — Launch an Ubuntu(22.04) T2 Large Instance
- Step 2   — Install Jenkins, Docker and Trivy. Create a Sonarqube Container using Docker.
- Step 3   — Create a TMDB API Key.
- Step 4   — Install Prometheus and Grafana On the new Server.
- Step 5   — Install the Prometheus Plugin and Integrate it with the Prometheus server.
- Step 6   — Email Integration With Jenkins and Plugin setup.
- Step 7   — Install Plugins like JDK, Sonarqube Scanner, Nodejs, and OWASP Dependency Check.
- Step 8   — Create a Pipeline Project in Jenkins using a Declarative Pipeline
- Step 9   — Install OWASP Dependency Check Plugins
- Step 10  — Docker Image Build and Push
- Step 11  — Deploy the image using Docker
- Step 12  — Kubernetes master and slave setup on Ubuntu (20.04)
- Step 13  — Access the Netflix app on the Browser.
- Step 14  — Terminate the AWS EC2 Instances.

# Step 1 ::
- Launch an Ubuntu(22.04) T2 Large Instance. You can create a new key pair or use an existing one. Enable HTTP and HTTPS settings in the Security Group and open all ports (not best case to open all ports but this is just for the project purpose).

![alt text](https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/Instances.png)

# Step 2 ::
- Install Jenkins, Docker and Trivy
- **2A — To Install Jenkins**
- Login to **Netflix-clone-project (Jenkins-Sonar)** server and Run these commands to Install Jenkins
```bash
vi jenkins.sh # Make sure you run the script in Root user (or) add in the Userdata while lauching the EC2 instance
```

```bash
#!/bin/bash
sudo apt update -y
#sudo apt upgrade -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
                  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
                  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
                              /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```


```bash
sudo chmod 777 jenkins.sh
./jenkins.sh    # this will installl jenkins
```

- Once Jenkins is installed, you will need to go to your AWS EC2 instance Security Group and open Inbound Port 8080, since Jenkins works on Port 8080.
- Now, copy the Public IP Address of this EC2 instance


```bash
<EC2_Public_IP_Address>:8080
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**2B — Install Docker ( Docker is used to run the Sonarqube container, By default sonar runs on the 9000 port )**
```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER   # In this case it is ubuntu "sudo usermod -aG docker ubuntu"
newgrp docker
docker ps
sudo chmod 777 /var/run/docker.sock
```
- After the docker installation, we create a sonarqube container (Remember to add 9000 ports in the security group).
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
docker ps
```

- Sonarqube will be running on port 9000
- To access this, **<EC2_Public_IP_Address>:9000**  # Login credentials for Sonarqube is admin / admin

**2C — Install Trivy ( Trivy is used for scanning the files present in the repository along with scanning the images )**
```bash
vi trivy.sh
sudo chmod +x trivy.sh
```
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```
```bash
sudo chmod +x trivy.sh
./trivy.sh  # Alternative command is -> sh trivy.sh
```

# Step 3 :: Create a TMDB (The Movie Database) API Key
- Now, we should create a TMDB API key
- Open a new tab in the Browser and search for TMDB
- Click on the first result, and navigate to TMDB Home page
- Click on the Login on the top right
- You need to create an account here if you don't have one and login to the account
- You should probably see the Home page
- Let's create an API key, By clicking on your profile and click on settings button
- Navigate to API section from the left side panel
- Now click on create
- Click on Developer and accept the terms and conditions
- Provide basic details and click on Submit button
- You will get your API Key and copy the API key and store it in some place

# Step 4 ::
- **Login to Prometheus-Grafana server** and perform the below actions
- Install Prometheus and Grafana on the above server
- Let's create a dedicated Linux USER sometimes called a system account for Prometheus. Having individual users for each service serves two main purposes:
  - It is a security measure to reduce the impact in case of an incident with the service.
  - It simplifies administration as it becomes easier to track down what resources belong to which service.
- To create a system user or system account, run the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
- where,
  - --system - Will create a system account.
  - --no-create-home - We don't need a home directory for Prometheus or any other system accounts in our case.
  - --shell /bin/false - It prevents logging in as a Prometheus user.
  - Prometheus - Will create a Prometheus user and a group with the same name.

- You can use the curl or wget command to download Prometheus ::

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```

- Then, we need to extract all Prometheus files from the archive.
```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```
- Usually, you would have a disk mounted to the data directory. For this tutorial, I will simply create a **/data** directory. Also, you need a folder for Prometheus configuration files.
```bash
sudo mkdir -p /data /etc/prometheus
```

- Now, let's change the directory to Prometheus and move some files.
```bash
cd prometheus-2.47.1.linux-amd64/
```

- First of all, let's move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.
```bash
sudo mv prometheus promtool /usr/local/bin/
```

- Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don't need to worry about it if you're just getting started.
```bash
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

- Finally, let's move the example of the main Prometheus configuration file.
```bash
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

- To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory.
```bash
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

- You can delete the archive and a Prometheus folder when you are done.
```bash
cd ..
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```

- Verify that you can execute the Prometheus binary by running the following command:
```bash
prometheus --version
```

- We're going to use some of these options in the service definition.
- We're going to use Systemd, which is a system and service manager for Linux operating systems. For that, we need to create a Systemd unit configuration file.
```bash
sudo vim /etc/systemd/system/prometheus.service
```

- COPY the below script into **Prometheus.service** file
```bash
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

- Let's go over a few of the most important options related to Systemd and Prometheus.
  - Restart - Configures whether the service shall be restarted when the service process exits, is killed, or a timeout is reached.
  - RestartSec - Configures the time to sleep before restarting a service.
  - User and Group - Are Linux user and a group to start a Prometheus process.
  - --config.file=/etc/prometheus/prometheus.yml - Path to the main Prometheus configuration file.
  - --storage.tsdb.path=/data - Location to store Prometheus data.
  - --web.listen-address=0.0.0.0:9090 - Configure to listen on all network interfaces. In some situations, you may have a proxy such as nginx to redirect requests to Prometheus. In that case, you would configure Prometheus to listen only on **localhost**.
  - --web.enable-lifecycle -- Allows to manage Prometheus, for example, to reload configuration without restarting the service.

- To automatically start the Prometheus after reboot, run enable, start the prometheus and check the status
```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status Prometheus
```

- Suppose you encounter any issues with Prometheus or are unable to start it. The easiest way to find the problem is to use the journalctl command and search for errors.
```bash
journalctl -u prometheus -f --no-pager
```

- Now we can try to access it via the browser. I'm going to be using the IP address of the Ubuntu server. You need to append port 9090 to the IP.
```bash
<Prometheus_Public_IP>:9090
```
- On the Prometheus Dashboard, if you go to targets, you should see only one - **Prometheus target**. It scrapes itself every 15 seconds by default.


**Install Node Exporter on Ubuntu 22.04** On the Prometheus-Grafana server
- Next, we're going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I'm not going to cover as deep as Prometheus.
- First, let's create a system user for Node Exporter by running the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```

- Use the wget command to download the binary.
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

- Extract the node exporter from the archive.
```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```

- Move binary to the /usr/local/bin.
```bash
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```

- Clean up, and delete node_exporter archive and a folder.
```bash
rm -rf node_exporter*
node_exporter --version
```

- **node_exporter runs on PORT 9100**

- Next, create a similar systemd unit file as done for Prometheus
```bash
sudo vim /etc/systemd/system/node_exporter.service
```

- COPY the below script into **node_exporter.service** file
```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target
```

- Replace Prometheus user and group to node_exporter, and update the ExecStart command.
- To automatically start the Node Exporter after reboot, enable the service and check the status.
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

- If you have any issues, check logs with journalctl
```bash
journalctl -u node_exporter -f --no-pager
```

- <p align="justify"> At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I'll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let's keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster. </p>
- To create a static target, you need to add job_name with static_configs.
```bash
sudo vim /etc/prometheus/prometheus.yml
```

- COPY the below script in the **prometheus.yml** file
```bash
  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```

- By default, Node Exporter will be exposed on port 9100.
- Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.
- Before, restarting check if the config is valid.
```bash
promtool check config /etc/prometheus/prometheus.yml
```

- Then, you can use a POST request to reload the config.
```bash
curl -X POST http://localhost:9090/-/reload
```

- Check the targets section
```bash
http://<prometheus-grafana_ip>:9090/targets
```


**Install Grafana on Ubuntu 22.04** on **Prometheus-Grafana** server
- To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus.
- First, let's make sure that all the dependencies are installed.
```bash
sudo apt-get install -y apt-transport-https software-properties-common
```

- Next, add the GPG key.
```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

- Add this repository for stable releases.
```bash
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```


- After you add the repository, update and install Garafana.
```bash
sudo apt-get update
sudo apt-get -y install grafana
```

- To automatically start the Grafana after reboot, enable the service and check the status.
```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

- Go to http://<prometheus-grafana_ip>:3000 and log in to the Grafana using default credentials. Default username and password is **admin**.

- To visualize metrics in Grafana dashboard, you need to add a Data source first
- Click Add data source and select Prometheus
- For the URL, enter http://<prometheus_public_ip>:9090 and click Save and test
- Let's add Dashboard for a better view
- Click on Import Dashboard by clicking '+' symbol on the top right corner and enter the code 1860 and click on Load button
- Select the Datasource and click on Import
- You will see this output with all the metrics on the screen


# Step 5 :: Install the Prometheus Plugin and Integrate it with the Prometheus server
- Let's Monitor **JENKINS SYSTEM**
- Jenkins should be UP and RUNNING
- Goto Manage Jenkins --> Plugins --> Available Plugins
- Search for **Prometheus** and install it
- Once that is done you will Prometheus is set to /Prometheus path in system configurations
- Nothing to change click on apply and save
- To create a static target, you need to add job_name with static_configs.

- **Go to Prometheus-Grafana server**
```bash
sudo vim /etc/prometheus/prometheus.yml
```

- Paste below code on the **prometheus.yml** file
```bash
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<netflix-clone-project_ip>:8080']
```

- Before, restarting check if the config is valid.
```bash
promtool check config /etc/prometheus/prometheus.yml
```

- Then, you can use a POST request to reload the config.
```bash
curl -X POST http://localhost:9090/-/reload
``

- Check the targets section
```bash
http://<prometheus-grafana_ip>:9090/targets
```

-You will see Jenkins is added to it
- Let's add Dashboard for a better view in Grafana
- Click On Dashboard --> '+' symbol --> Import Dashboard
- Use Id 9964 and click on load
- Select the data source and click on Import
- Now you will see the Detailed overview of Jenkins


# Step 6 :: Install Email Integration With Jenkins and Plugin Setup
- Install **Email Extension Plugin** in Jenkins
- Go to your Gmail and click on your profile
- Then click on Manage Your Google Account --> click on the security tab on the left side panel and navigate to "How you sign in to Google" section
- 2-step verification should be enabled
- Search for the **app** in the search bar you will get **App passwords** option, click on that option
- Copy the password that is shows on the screen (Mixed characters string)
- Once the plugin is installed in Jenkins, click on manage Jenkins --> configure system, navigate to the E-mail Notification section and configure the below details:
  - SMTP server --> smtp.gmail.com
  - Click on **Advanced** option
    - Check the box of **Use SMTP Authentication** and **Use SSL**
    - User Name --> <provide_your_email_address>
    - Password ---> <provide_password>
    - SMTP Port --> 465
    - Charset --> UTF-8
- Click on Apply and save.
- Click on Manage Jenkins --> credentials --> System --> Global credentials, and add your mail username and GENERATED password (From the App passwords)
- This is to just verify the mail configuration
- Now under the Extended E-mail Notification section configure the details as below.
  - SMTP server --> smtp.gmail.com
  - SMTP Port --> 465
  - Click on **Advanced** option
    - From the **Credentails** dropdown, select the email which is created above (created in the Step 6)
    -  Check the box of **Use SSL**
  - Default Content Type --> HTML (text/html)
  - Click on **Default Triggers** option
    - Check the box of **Always** and **Failure-Any**
- Click on Apply and Save

- Next, we will log in to Jenkins and start to configure our Pipeline in Jenkins

# Step 7 :: Install Plugins like JDK, Sonarqube Scanner, NodeJs, OWASP Dependency Check
- **7A — Install Plugin**
- Goto Manage Jenkins --> Plugins --> Available Plugins --> Install below plugins
  - Eclipse Temurin Installer (Install without restart)
  - SonarQube Scanner (Install without restart)
  - NodeJs Plugin (Install Without restart)


- **7B — Configure Java and Nodejs in Global Tool Configuration**
- Goto Manage Jenkins --> Tools --> Install JDK(17) and NodeJs(16) --> Click on Apply and Save

- **7C — Create a Job**
- create a job as Netflix Name, select pipeline and click on Ok.

# Step 8 :: Configure Sonar Server in Manage Jenkins
- Grab the Public IP Address of your **Netflix-clone-project EC2 Instance**, Sonarqube works on Port 9000, so <Netflix_clone_project_Public_IP>:9000.
- Goto your Sonarqube Server.
- Click on Administration --> Security --> Users --> Click on Tokens and Update Token --> Give it a name --> and click on Generate Token
- Click on update Token
- Create a token with a name and generate
- Copy Token
- Goto Jenkins Dashboard --> Manage Jenkins --> Credentials --> Add Secret Text
- Now, go to Dashboard --> Manage Jenkins --> System and Add the details of Name, Server URL and Serber authentication token
- Click on Apply and Save
- The **Configure System option** is used in Jenkins to configure different server
- **Global Tool Configuration** is used to configure different tools that we install using Plugins
- We will install a sonar scanner in the tools
- In the Sonarqube Dashboard add a quality gate also, Administration--> Configuration-->Webhooks
- Click on Create
- Add details

```bash
#in url section of quality gate
<http://Netflix_clone_project_Public_IP:8080>/sonarqube-webhook/
```
- Let's go to our Pipeline and add the script in our Pipeline Script.
```bash
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'rutik@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

```

- Click on Build now, and we should be able to see the pipeline which is started.
- To see the report, you can go to Sonarqube Server and go to Projects.
- You can see the report has been generated and the status shows as passed. You can see that there are 3.2k lines it scanned. To see a detailed report, you can go to issues.

# Step 9 :: Install OWASP Dependency Check Plugins
- Goto Dashboard --> Manage Jenkins --> Plugins --> OWASP Dependency-Check. Click on it and install it without restart.
- First, we configured the Plugin and next, we had to configure the Tool
- Goto Dashboard --> Manage Jenkins --> Tools
  - Add the Name for **Add Dependency-Check**
  - Check the box for **Install automatically**
  - Provide the Version
- Click on Apply and Save.
- Now go configure --> Pipeline and add this stage to your pipeline and build.

```bash
stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
```

# Step 10 :: Docker Image Build and Push
- We need to install the Docker tool in our system, Goto Dashboard --> Manage Plugins --> Available plugins --> Search for Docker and install these plugins
  - Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step
- And click on install without restart
- Now, goto Dashboard --> Manage Jenkins --> Tools -->
  - Add **Name** for the Docker section
  - Check the box for **Install automatically**
  - Docker version --> latest
- Add DockerHub Username and Password under Global Credentials
- Add this stage to Pipeline Script

```bash
stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=Aj7ay86fe14eca3e76869b92 -t netflix ."
                       sh "docker tag netflix sevenajay/netflix:latest "
                       sh "docker push sevenajay/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/netflix:latest > trivyimage.txt" 
            }
        }
```


- When you log in to Dockerhub, you will see a new image is created with the name **netflix**
- Now Run the container to see if the game coming up or not by adding the below stage

```bash
stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 sevenajay/netflix:latest'
            }
        }
```


# Step 11 :: Kuberenetes Setup
- Connect your machines to Putty or Mobaxterm (Windows) or Terminal (MacOS)
- Take-Two Ubuntu 20.04 EC2 instances one for K8s master and the other one for K8sworker.
- Install Kubectl on Jenkins machine also.
- Kubectl is to be installed on Jenkins also

```bash
sudo apt update
sudo apt install curl -y
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```


- Part 1 ----------Master Node------------
```bash
sudo hostnamectl set-hostname K8s-Master
```

- ----------Worker Node------------
```bash
sudo hostnamectl set-hostname K8s-Worker
```

- Part 2 ------------Both Master & Node ------------
```bash
sudo apt-get update 

sudo apt-get install -y docker.io
sudo usermod –aG docker Ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl

sudo snap install kube-apiserver
```

- Part 3 --------------- Master ---------------
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# in case your in root exit from it and run below commands
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

- ----------Worker Node------------
```bash
sudo kubeadm join <master-node-ip>:<master-node-port> --token <token> --discovery-token-ca-cert-hash <hash>
```

- Copy the config file to Jenkins master or the local file manager and save it
- Copy it and save it in documents or another folder save it as secret-file.txt
- Note: create a secret-file.txt in your file explorer save the config in it and use this at the kubernetes credential section.
- Install Kubernetes Plugin, Once it's installed successfully
  - K8s
  - K8s Credentials
  - K8s Client API
  - K8s CLI
  - K8s Credentials Provider
- Goto manage Jenkins --> manage credentials --> Click on Jenkins global --> add credentials


**Install Node_exporter on both master and worker**
- Let's add Node_exporter on Master and Worker to monitor the metrics
- First, let's create a system user for Node Exporter by running the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```


- Use the wget command to download the binary.
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

- Extract the node exporter from the archive.
```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```
- Move binary to the /usr/local/bin.
```bash
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```

- Clean up, and delete node_exporter archive and a folder.
```bash
rm -rf node_exporter*
```

- Next, create a similar systemd unit file.
```bash
sudo vim /etc/systemd/system/node_exporter.service
```

- COPY the below script in **node_exporter.service** file
```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target
```

- Replace Prometheus user and group to node_exporter, and update the ExecStart command.
- To automatically start the Node Exporter after reboot, enable the service.
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

- If you have any issues, check logs with journalctl
```bash
journalctl -u node_exporter -f --no-pager
```

- At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. For this project, let's keep it simple and keep adding static targets.
- To create a static target, you need to add job_name with static_configs.
- Go to **Prometheus-Grafana** server
```bash
sudo vim /etc/prometheus/prometheus.yml
```

- COPY the following script in **prometheus.yml**
```bash
  - job_name: node_export_masterk8s
    static_configs:
      - targets: ["<master-ip>:9100"]

  - job_name: node_export_workerk8s
    static_configs:
      - targets: ["<worker-ip>:9100"]
```
- By default, Node Exporter will be exposed on port 9100.
- Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.
- Before, restarting check if the config is valid.
```bash
promtool check config /etc/prometheus/prometheus.yml
```

- Then, you can use a POST request to reload the config.
```bash
curl -X POST http://localhost:9090/-/reload
```

- Check the targets section
```bash
http://<prometheus-grafana_public_ip>:9090/targets
```

- Final step to deploy on the Kubernetes cluster
```bash
stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }
```

- In the Kubernetes cluster(master) give this command
```bash
kubectl get all 
kubectl get svc #use anyone
```

# STEP 12:: Access from a Web browser with
- <public-ip-of-slave:service port>


# Project Screenshots (Output):
<p align="center">
<img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/prometheus.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/sonarqube.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/grafana.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/jenkins-pipeline.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/grafana1.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/grafana2.png" />
</p>
<p align="center">
  <img src="https://github.com/kypamourya27/Netflix-Clone-CI-CD-pipeline-with-Monitoring-Alerting/blob/main/assets/netflix.png" />
</p>



# Step 13::
- Terminate all the instances created as part of this project


# Project Reference ::
- https://youtu.be/pbGA-B_SCVk?feature=shared
- https://mrcloudbook.hashnode.dev/devsecops-netflix-clone-ci-cd-with-monitoring-email
