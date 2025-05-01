# jfrog-nexus

# Upload artifacts from debian based machine into jfrog using jfrog cli.  
Pre-condition : project and the repository is created in jfrog.  

Install jfrog CLI:  

curl -fL https://getcli.jfrog.io | sh  
sudo mv jfrog /usr/local/bin  
sudo mv jfrog /usr/local/bin  

Configure jfrog server:  
Add a new JFrog CLI configuration named my-server that connects to the Artifactory URL using access token  
sudo jfrog config add my-server  --url=<jfrog url> --access-token= <jfrog access token>  
Ex:   
jfrog config add my-server  --url=https://triale6ujpm.jfrog.io/  --access-token=eyJ2ZXIiOiI  


Upload artifacts:  
jfrog rt u <"Jar file"> <Repo-name>  
Ex:  
jfrog rt u   "maven-calc-jenkins-1.0-SNAPSHOT.jar" calculator-local/  

Verify file uploaded:  
jfrog rt s <repo>-name>/<artifact-name>  
jfrog rt s caculator-local/maven-calc-jenkins-1.0-SNAPSHOT.jar  


# Upload artifacts from GitHub Actions pipeline  
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

# Upload artifacts from Jenkins pileline  
