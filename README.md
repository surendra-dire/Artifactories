# Artifactory & Scanning  


## Upload Artifacts  

_This text is underlined_

### _Using Jfrog cli_ 
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
### _Using GitHub Actions_ 
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

# _Using Jenkins pipeline_



# Jfrog vs Nexus

## 🔄 JFrog Artifactory vs Sonatype Nexus

| # | Category                      | JFrog Artifactory                                                                 | Sonatype Nexus Repository                                             |
|---|-------------------------------|-----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 1 | 🔧 Artifact Format Support     | ✅ Supports 30+ formats (Docker, Go, Helm, Conan, npm, NuGet, etc.)               | ⚠️ Supports ~8 formats (Maven, npm, NuGet, Docker, PyPI, etc.)       |
| 2 | 📦 Build Metadata & Promotion | ✅ Native support via CLI/API (build-info, promotion across repos)                | ❌ No native support (manual scripting or Nexus Pro required)        |
| 3 | 🔐 Security & License Scanning| ✅ Built-in via **JFrog Xray** (CVEs, license compliance, policy enforcement)     | ⚠️ Only with **Nexus IQ** (separate licesne)                         |
| 4 | ☁️ SaaS Availability          | ✅ Available as **fully managed SaaS (Artifactory Cloud)**                        | ❌ Self-hosted only – no SaaS option                                 |
| 5 | 🧪 CI/CD Integration & Automation| ✅ Deep integration with CI/CD tools (GitHub Actions, Jenkins, GitLab, etc.)    | ⚠️ Basic REST API & CLI; limited in OSS version                      |


## 🔍 JFrog Xray vs Trivy vs Nexus 

| Capability                      | **JFrog Xray**                             | **Trivy**                                      | **Nexus**                         |
|--------------------------------|--------------------------------------------|------------------------------------------------|---------------------------------------------|
| Scan Docker images             | ✅ Yes (when stored in Artifactory)        | ✅ Yes (any source, including local)           | ✅ Yes (via Nexus IQ Scanner integration)    |
| Scan Terraform / K8s YAML      | ❌ No                                      | ✅ Yes                                         | ❌ No (not natively supported)               |
| Scan Git repositories          | ❌ No                                      | ✅ Yes                                         | ❌ No (scans dependencies, not code repos)   |
| Requires central repository    | ✅ Yes (Artifactory)                       | ❌ No                                          | ✅ Yes (Nexus Repository or IQ Server)       |
| Local/standalone usage         | ❌ No (requires JFrog setup)               | ✅ Yes (CLI tool, easy install)               | ⚠️ Limited (requires IQ CLI + server setup)  |
| Ecosystem coverage             | Docker, Maven, npm, PyPI, etc.             | Docker, OS packages, IaC, Git repos, etc.     | Maven, npm, PyPI, NuGet, Go, etc.           |
| Policy enforcement             | ✅ Centralized via JFrog Platform          | ❌ Manual (via CI scripting or exit codes)     | ✅ Yes (via Nexus IQ policies)               |
| License compliance checks      | ✅ Yes                                     | ⚠️ Basic (license info only)                  | ✅ Yes                                       |
| Reporting UI                   | ✅ JFrog Web UI                             | ⚠️ Basic CLI / JSON / HTML output             | ✅ Yes (IQ Server dashboards)                |
| Open-source                    | ❌ No (Commercial)                         | ✅ Yes (Apache 2.0)                            | ❌ No (Commercial)                           |
| Ideal for                      | Enterprises using Artifactory              | Devs & teams needing fast, lightweight scans   | Teams using Nexus for repo + security       |

---


## 🔍 Use cases

### ✅ Use JFrog Xray when:
- Best for teams using Artifactory who need deep security and compliance scanning.
- You want license, security, and operational risk scanning in one place.

### ✅ Use Nexus Lifecycle when:
- Best for organizations managing open-source libraries with policy-driven enforcement
- You need license and security scanning with detailed dashboards.

### ✅ Use Trivy when:
- You want a fast, free CLI scanner.
- You need to scan Docker, OS packages, or IaC files in CI/CD.
- You want flexibility without depending on a central platform.

- # Repository Types in JFrog Artifactory vs Sonatype Nexus

| Feature                     | JFrog Artifactory                      | Sonatype Nexus                      |
|-----------------------------|----------------------------------------|-------------------------------------|
| **Local/hosted**         | Stores internal artifacts              | Stores internal artifacts           |
| **Remote/Proxy**         | Proxies external repositories (cache for external repositories)          | Proxies external repositories (cache for external repositories)      |
| **Virtual/Group**        | Aggregates multiple repositories (group multiple repositories into a single URL)      | Aggregates multiple repositories (group multiple repositories into a single URL)    |
| **Distribution**         | Optimized for distributing packages    | Not available in the same form (typically handled by proxy/group) |
| **Enterprise**           | Advanced security, HA, scalability, advanced distribution | Generally lighter, more simplified, good for basic artifact management |
| **Cost**                 | Paid version with advanced features (Free for basic use) | Free OSS version, enterprise version has additional features |

**Java maven repositories:**
Local (jfrog) :  Can hadle both Release and snapshot version
Hosted (nexus):  Release version (immutable - can not override), Snapshot version (Mutable - can be overwridden)


## 🔍 Vulnerability Scanning Tools Comparison

**JFrog Xray**, **Nexus IQ**, and **Trivy** use cases

| Tool           | When to Choose                                                                                                         | Ideal For                                   | License Type                 |
|----------------|-------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|------------------------------|
| **JFrog Xray** | Best for teams using Artifactory who need deep security and compliance scanning | Enterprise DevSecOps with Artifactory         | Commercial (Free tier available) |
| **Nexus IQ**   |  Best for organizations managing open-source libraries with policy-driven enforcement | Enterprises managing open-source components  | Commercial (Free tier available) |
| **Trivy**      |Best for containerized environments  **containers (Docker, Kubernetes)** needing fast scanner, simple, open-source scanning in **CI/CD**. | DevOps, Cloud-native, open-source projects   | Open-source (Free)           |

---

