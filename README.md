# jfrog-nexus

# Upload artifacts from debian based machine into jfrog using jfrog cli.  
Pre-condition : project and the repository is created in jfrog.  

Install jfrog CLI:  
```
curl -fL https://getcli.jfrog.io | sh  
sudo mv jfrog /usr/local/bin  
sudo mv jfrog /usr/local/bin  
```
Configure jfrog server:  
Add a new JFrog CLI configuration named my-server that connects to the Artifactory URL using access token  
```
sudo jfrog config add my-server  --url=<jfrog url> --access-token= <jfrog access token>  
Ex:   
jfrog config add my-server  --url=https://triale6ujpm.jfrog.io/  --access-token=eyJ2ZXIiOiI  
```
Upload artifacts:  
```
jfrog rt u <"Jar file"> <Repo-name>  
Ex:  
jfrog rt u   "maven-calc-jenkins-1.0-SNAPSHOT.jar" calculator-local/  
```
Verify file uploaded:  
```
jfrog rt s <repo>-name>/<artifact-name>  
jfrog rt s caculator-local/maven-calc-jenkins-1.0-SNAPSHOT.jar
```
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



# Jfrog vs Nexus

## 🔄 JFrog Artifactory vs Sonatype Nexus – Key Differences

| # | Category                      | JFrog Artifactory                                                                 | Sonatype Nexus Repository                                             |
|---|-------------------------------|-----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 1 | 🔧 Artifact Format Support     | ✅ Supports 30+ formats (Docker, Go, Helm, Conan, npm, NuGet, etc.)               | ⚠️ Supports ~8 formats (Maven, npm, NuGet, Docker, PyPI, etc.)       |
| 2 | 📦 Build Metadata & Promotion | ✅ Native support via CLI/API (build-info, promotion across repos)                | ❌ No native support (manual scripting or Nexus Pro required)        |
| 3 | 🔐 Security & License Scanning| ✅ Built-in via **JFrog Xray** (CVEs, license compliance, policy enforcement)     | ⚠️ Only with **Nexus IQ** (separate paid product)                    |
| 4 | ☁️ SaaS Availability          | ✅ Available as **fully managed SaaS (Artifactory Cloud)**                        | ❌ Self-hosted only – no SaaS option                                 |
| 5 | 🧪 CI/CD Integration & Automation| ✅ Deep integration with CI/CD tools (GitHub Actions, Jenkins, GitLab, etc.)    | ⚠️ Basic REST API & CLI; limited in OSS version                      |

## 🔍 JFrog Xray vs Trivy: Feature Comparison

| Capability                      | JFrog Xray                                | Trivy                                      |
|--------------------------------|-------------------------------------------|--------------------------------------------|
| Scan Docker images             | ✅ Yes (when stored in Artifactory)       | ✅ Yes (any source, including local)       |
| Scan Terraform / K8s YAML      | ❌ No                                     | ✅ Yes                                     |
| Scan Git repositories          | ❌ No                                     | ✅ Yes                                     |
| Needs artifacts in Artifactory| ✅ Yes                                     | ❌ No                                      |
| Local/standalone usage         | ❌ No (requires JFrog setup)              | ✅ Yes (lightweight CLI tool)              |
| Ecosystem coverage             | Docker, Maven, npm, PyPI, etc.            | Docker, OS packages, IaC, Git repos, etc. |
| Policy enforcement             | ✅ Centralized in JFrog Platform          | ❌ Manual via CI scripting                 |
| Licensing & compliance checks  | ✅ Yes

## 🔍 Use cases

### ✅ Use JFrog Xray when:
- You are using JFrog Artifactory and want integrated artifact security.
- You need enterprise-grade policy enforcement, license compliance, and audit trails such as health care industry.
- You need to block vulnerable artifacts during promotion or release.

### ✅ Use Trivy when:
- You want fast, free, local or CI-based vulnerability scanning.
- You need to scan IaC files (Terraform, Kubernetes YAML).
- You want a lightweight tool for scanning Docker images or code repositories.
