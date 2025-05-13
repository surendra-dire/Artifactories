# Nexus ( Nexus Repository Manager)  
**Nexus** provides a centralized platform for managing software dependencies and build artifacts.  Nexus Repository Manager is a software repository manager developed by Sonatype. It is available in two different editions:  

**Nexus Repository OSS (Open Source Software)/Community Edition (CE)**: A free, open-source edition offering essential functionality. Ideal for small teams and organizations that need core repository features without incurring licensing costs. It has embedded H2 database support. Nexus Repository CE includes usage limits of 200,000 requests per day and 100,000 components.  

**Nexus Repository Pro**: A commercial version with advanced features and enterprise level support. Supports PostgreSQL/H2/Oracle databases.

## Upload Artifacts  

### A) _Using Jfrog cli [manully)_ 
Pre-condition : project and the repository is created in jfrog.  

**1) Install jfrog CLI:** 
```   
curl -fL https://getcli.jfrog.io | sh  
sudo mv jfrog /usr/local/bin  
jfrog --version
```  
**2) Configure jfrog server:** 
Add a new JFrog CLI configuration named my-server that connects to the Artifactory URL using access token  
```
sudo jfrog config add my-server  --url=<jfrog url> --access-token= <jfrog access token>  
Ex:   
jfrog config add my-server  --url=https://triale6ujpm.jfrog.io/  --access-token=eyJ2ZXIiOiI  
```
**3) Upload artifacts:**  
```
jfrog rt u <"Jar file"> <Repo-name>  
Ex:  
jfrog rt u   "maven-calc-jenkins-1.0-SNAPSHOT.jar" calculator-local/  
```
**4) Verify file uploaded:**  
```
jfrog rt s <repo>-name>/<artifact-name>  
jfrog rt s caculator-local/maven-calc-jenkins-1.0-SNAPSHOT.jar
```
### B) _Using GitHub Actions - CLI_ 
1. Jfrog: Create project--> Repository --> permissions
2. Install and configure the JFrog CLI
3. Upload jar/war to Artifactory
```
     # Set up JFrog CLI (this installs & configures it)
      - name: Set up JFrog CLI
        uses: jfrog/setup-jfrog-cli@v3
        with:
          version: latest
        env:
          JF_URL: ${{ secrets.JFROG_URL }}
          JF_ACCESS_TOKEN: ${{ secrets.JFROG_TOKEN }}

      # Use 'jf' inside a single shell script
      - name: Upload JAR to Artifactory with Commit ID and Date
        shell: bash
        run: |
          echo "Uploading artifact..."
          COMMIT_ID=$(echo $GITHUB_SHA | cut -c1-7)
          DATE=$(date +'%Y-%m-%d')
          ARTIFACTORY_PATH="calculator-local/test/com/${DATE}-${COMMIT_ID}/"
          echo "Uploading to: $ARTIFACTORY_PATH"
          jf rt upload "target/*.jar" "$ARTIFACTORY_PATH" --flat=true

      # or simply
      # - name: Upload JAR to Artifactory
          #   run: |
          #     jf rt upload "target/*.jar" "calculator-local/test/com/"
```
### C) _Using Jenkins pipeline - CLI_ 

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

### D) _Using Jenkins pipeline - Artifactory Jenkins plugin_ 
1. Configures the Artifactory server.Sets the connection details for Artifactory (rtServer)
2. Upload Artifacts to Artifactory (rtUpload)
3. 
```
pipeline {
    agent any  
    tools {
        maven 'maven 3.8.7'  
    }
    environment {
        //JFrog credentials ID
        JFROG_CREDENTIALS_ID = 'jfrog-instance-id'
        
        // Build version for artifact tracking
        BUILD_VERSION = "${env.BUILD_NUMBER}"
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
        stage('Setup Artifactory') {
            steps {
                script {
                    // Configure Artifactory server
                    def server = Artifactory.server('jfrog-instance-id')
                    env.ARTIFACTORY_URL = server.url
                    echo "Artifactory URL: ${env.ARTIFACTORY_URL}"
                    
                    rtServer (
                        id: 'my-artifactory-server',                       
                        url: "${env.ARTIFACTORY_URL}",                          // Artifactory URL
                        credentialsId: "${JFROG_CREDENTIALS_ID}"                // Jenkins credentials - fixed the variable name
                    )
                }
            }
        }
        
        stage('Upload to Artifactory') {
            steps {
                script {
                    def buildInfo = rtBuildInfo()  
                    def currentDate = new Date().format("yyyy-MM-dd_HH-mm-ss")
                    echo "Current date: ${currentDate}"

                    // You can also set it as an environment variable if needed
                    env.BUILD_DATE = currentDate
                    
                    // Upload artifacts to Artifactory
                    rtUpload (
                        serverId: 'my-artifactory-server',     // Server ID defined in rtServer
                        spec: """{
                            "files": [
                                {
                                    "pattern": "target/*.jar",                                                
                                    "target": "calculator-local/${BUILD_VERSION}/",  
                               
                                    "props": "build.date=${env.BUILD_DATE};build.number=${BUILD_NUMBER}"  
                                }
                            ]
                        }""",
                        buildInfo: buildInfo
                    )
                    
                    // Publish the build info to Artifactory
                    rtPublishBuildInfo (
                        serverId: 'my-artifactory-server',
                        buildInfo: buildInfo
                    )
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
        always {
            // Clean workspace after build (optional)
            cleanWs()
        }
    }
}
```

