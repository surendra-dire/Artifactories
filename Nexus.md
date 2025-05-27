# Nexus ( Nexus Repository Manager)  
**Nexus** provides a centralized platform for managing software dependencies and build artifacts.  Nexus Repository Manager is a software repository manager developed by Sonatype. It is available in two different editions:  

**Nexus Repository OSS (Open Source Software)/Community Edition (CE)**: A free, open-source edition offering essential functionality. Ideal for small teams and organizations that need core repository features without incurring licensing costs. It has embedded H2 database support. Nexus Repository CE includes usage limits of 200,000 requests per day and 100,000 components.  

**Nexus Repository Pro**: A commercial version with advanced features and enterprise level support. Supports PostgreSQL/H2/Oracle databases.

## Upload Artifacts  

### A) _Using GitHub Actions_
1. Nexus: Create project--> Repository --> user permissions
2. Authenitcate & upload the jar/war to nexus repository
```
name: Build-Nexus

on:
  workflow_dispatch:
  
jobs:
  build:
    runs-on: [self-hosted, label1]
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'adopt'
          cache: maven

      - name: Cache Maven Dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Build with Maven
        run: mvn clean install

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: jar
          path: target/*.jar

      # Step 1: Get SNAPSHOT/Release version from pom.xml
      - name: Get version info
        id: version-info
        run: |
          VERSION=$(mvn help:evaluate -Dexpression=project.version -DforceStdout -q \
            | grep -E '^[0-9]+\.[0-9]+.*$' \
            | sed 's/\x1B\[[0-9;]*[a-zA-Z]//g' \
            | tr -d '\r')
      
          echo "Cleaned Maven version: '$VERSION'"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
      
          if [[ "$VERSION" == *-SNAPSHOT ]]; then
            echo "is_snapshot=true" >> "$GITHUB_OUTPUT"
          else
            echo "is_snapshot=false" >> "$GITHUB_OUTPUT"
          fi
      
      # Step 2: Set repository name (snapshot/release)
      - name: Set repository name to snapshot
        if: ${{ steps.version-info.outputs.is_snapshot == 'true' }}
        run: echo "REPO_NAME=${{ secrets.NEXUS_SNAPSHOT_REPO }}" >> $GITHUB_ENV
      
      - name: Set repository name to release
        if: ${{ steps.version-info.outputs.is_snapshot != 'true' }}
        run: echo "REPO_NAME=${{ secrets.NEXUS_RELEASE_REPO }}" >> $GITHUB_ENV
      
      # STEP 3: Configure maven to connect to a nexus remote repository
      - name: Configure Maven for Nexus
        uses: s4u/maven-settings-action@v2
        with:
          servers: '[{ "id": "nexus", "username": "${{ secrets.NEXUS_USERNAME }}", "password": "${{ secrets.NEXUS_PASSWORD }}" }]'
      
      # STEP 4: Deploy JAR
      - name: Deploy main JAR to Nexus
        run: |
          JAR_FILE=$(find target -type f -name "*.jar" | head -n 1)
          echo "Found JAR: $JAR_FILE"
          echo "Target URL: ${{ secrets.NEXUS_SERVER_URL }}/repository/$REPO_NAME/"

          mvn deploy:deploy-file \
            -Dfile="$JAR_FILE" \
            -DgroupId=com.example \
            -DartifactId=maven-calc-jenkins \
            -Dversion=${{ steps.version-info.outputs.version }} \
            -Dpackaging=jar \
            -DrepositoryId=nexus \
            -Durl="${{ secrets.NEXUS_SERVER_URL }}/repository/$REPO_NAME/" \
            -DgeneratePom=false 
     
```
### B) _Using Jenkins pipeline_

1. First update the pom.xml file and add distribution management section having nexus repository url
```
<distributionManagement>
    <repository>
        <id>my-nexus-release</id>
        <url>http://192.168.178.94:8082/repository/Calc-maven-release-repo/</url>
    </repository>
    <snapshotRepository>
        <id>my-nexus-snapshot</id>
        <url>http://192.168.178.94:8082/repository/Calc-maven-hosted-repo/</url>
    </snapshotRepository>
</distributionManagement>
```
2. Manage the settings.xml file for user details [Managed Jenkins --> Managed files --> Add a new config]
   Note: Install the Config File Provider Plugin for Managed Jenkins. Also ensure repository id should match with from distributionManagement.
```
   <server>
      <id>my-nexus-snapshot</id>
      <username>admin</username>
      <password>admin</password>
    </server>
     <server>
      <id>my-nexus-release</id>
      <username>admin</username>
      <password>admin</password>
    </server>
```
3. Use Pipeline Syntax --> Select step **withMaven: Provide Maven environment**. Select maven and settings.xml global file and generate syntax.
   Add maven deploy.
```
steps {
	withMaven(globalMavenSettingsConfig: 'global-settings-id', jdk: '', maven: 'maven 3.8.7', mavenSettingsConfig: '', traceability: true) {
	     sh 'mvn deploy'
	}
```
4.  End to end pipeline
```
 pipeline {
    agent any  

    tools {
        maven 'maven 3.8.7'   // Ensure this is configured in Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning the repository...'
                git url: 'https://github.com/surendra-dire/maven-calc-jenkins.git', branch: 'main'
            }
        }

        stage('Build') {
            steps {
                echo 'Building the project...'
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test -Dsurefire.timeout=300'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn package'
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo 'Archiving the JAR...'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }

        stage('Upload to Nexus repository') {
            steps {
                echo 'Deploying artifacts in Nexus ...'
                withMaven(globalMavenSettingsConfig: '', jdk: '', maven: '', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        } 
    }

    post {
        success {
            echo 'Build completed successfully!'
        }
        failure {
            echo 'Build failed.'
        }
    }
}
```
5.  
6.  
 

## Nexus  -Installation (Ubuntu)   

**update package list**  
```
sudo apt update
sudo apt install openjdk-11-jdk
or
wget https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.17%2B8/OpenJDK11U-jdk_x64_linux_hotspot_11.0.17_8.tar.gz
tar xvfz OpenJDK11U-jdk_x64_linux_hotspot_11.0.17_8.tar.gz
mv jdk-11.0.17+8 jdk-11
mv jdk-11 /opt/java/jdk-11
```
**Download and extract the nexus (Go with current LTS version)**
```
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-3.80.0-06-linux-x86_64.tar.gz
sudo tar xvfz nexus-*.tar.gz
sudo mv nexus-3.80.0-06 nexus
```
**Configure the nexus to run as service. Create user and change ownership.**  
```
sudo useradd -r -m -s /bin/bash nexus
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
```

**Configure Nexus to Run as a Service**
```
sudo nano /etc/systemd/system/nexus.service
[Unit]
Description=Nexus Repository Manager
Documentation=http://help.sonatype.com/
After=network.target

[Service]
Environment="JAVA_HOME=/opt/java/jdk-11"
User=nexus
Group=nexus
Type=forking
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

# Reload Daemon service and enable prometheus to start post reboot
sudo systemctl daemon-reload
sudo systemctl enable nexus
sudo systemctl start nexus
sudo systemctl status nexus

# Ensure port 9090 is open in firewall
```
## Manually upload artifacts to Nexus from windows
**Configure the .m2/settings.xml**
```
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>admin</password>
    </server>
  </servers>
</settings>
```
**Execute maven deploy**
```
mvn deploy:deploy-file  -Dfile=maven-calc-jenkins-1.0-SNAPSHOT.jar   -DgroupId=com.example    -DartifactId=maven-calc-Jenkins    -Dversion=3.3-SNAPSHOT    -Dpackaging=jar    -DgeneratePom=true    -DrepositoryId=nexus  -Durl=http://63.35.231.45:8081/repository/Calc-maven-hosted-repo/
```

## Additonal notes  
**log file location:** /opt/nexus/sonatype-work/nexus3/log
**Default password:** cat /opt/sonatype-work/nexus3/admin.password   
**Check logs:** sudo journalctl -u nexus.service -f   [or] ps aux | grep nexus  
**Nexus log file location:**   
**Run nexus manually** sudo -u nexus /opt/nexus/bin/nexus run    




