# jfrog-nexus

# Upload artifacts from debian based machine into jfrog using jfrog cli.
Pre-condition : project and the repository is created in jfrog.

Install jfrog CLI:

curl -fL https://getcli.jfrog.io | sh
sudo mv jfrog /usr/local/bin
sudo mv jfrog /usr/local/bin

Configure jfrog server:

sudo jfrog config add my-server  --url=<jfrog url> --access-token= <jfrog access token>


Upload artifacts:
jfrog rt u <"Jar file"> <Repo-name>
jfrog rt u   "maven-calc-jenkins-1.0-SNAPSHOT.jar" calculator-local/

