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

1. Setup artifactory credentials.
2. Install the JFrog Artifactory plugin in Jenkins and configure the Artifactory server URL along with the required credentials.  
3. Install the JFrog CLI, either manually on the build server or dynamically within the Jenkins pipeline.  
4. Configure the JFrog CLI with Artifactory connection details (server ID, URL, username, and password/API key).   
Note: The JFrog CLI acts as a client that communicates with Artifactory. Artifactory is a secured artifact repository.  
	1) Where to send requests (Artifactory URL).  
	2) Who is making the request (Username).
	3) How to verify the user (Password or token)
5. Use the JFrog CLI commands in the pipeline to upload artifacts to the configured Artifactory repository.  
```
pipeline {
    agent any  

    tools {
        maven 'maven 3.8.7'  								          // Ensure this is configured in Jenkins
    }

    environment {
        MAVEN_OPTS = "-Xmx1024m"
        ARTIFACTORY_URL = 'https://triale6ujpm.jfrog.io/artifactory'				     // From plugin configuration
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning the repository...'
                git url: 'https://github.com/surendra-dire/maven-calc-jenkins.git', branch: 'main'      // Update with your repo
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
                sh 'mvn test'
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

        stage('Upload to Artifactory') {
            steps {
                echo 'Uploading artifacts to Artifactory...'
                script {
                    withCredentials([usernamePassword(					 // From Jenkins credentials
                        credentialsId: 'jfrog-instance-id',
                        usernameVariable: 'ARTIFACTORY_USER',
                        passwordVariable: 'ARTIFACTORY_PASSWORD'
                    )]) {
                        sh '''
                            if ! jfrog config show my-server > /dev/null 2>&1; then
                                jfrog config add my-server \
                                    --artifactory-url=$ARTIFACTORY_URL \
                                    --user=$ARTIFACTORY_USER \
                                    --password=$ARTIFACTORY_PASSWORD \
                                    --interactive=false
                            fi

                            jfrog rt upload "target/*.jar" "calculator-local/com/mycompany/your-artifact/"    // Update with your repository
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build completed successfully!'
        }
        failure {
            echo '❌ Build failed.'
        }
    }
}
```


```
## Nexus  -Installation (Ubuntu)   

# update package list
sudo apt update
sudo apt install openjdk-11-jdk
or
wget https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.17%2B8/OpenJDK11U-jdk_x64_linux_hotspot_11.0.17_8.tar.gz
tar xvfz OpenJDK11U-jdk_x64_linux_hotspot_11.0.17_8.tar.gz
mv jdk-11.0.17+8 jdk-11
mv jdk-11 /opt/java/jdk-11

# Download and extract the nexus (Go with current LTS version)
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-3.80.0-06-linux-x86_64.tar.gz
sudo tar xvfz nexus-*.tar.gz
sudo mv nexus-3.80.0-06 nexus

# Configure the nexus to run as service. Create user and change ownership.  
sudo useradd -r -m -s /bin/bash nexus
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work

# Configure Nexus to Run as a Service
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
