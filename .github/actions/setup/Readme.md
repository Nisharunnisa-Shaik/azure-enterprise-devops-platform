This guide shows how to install Maven from the public Apache.
Install Maven from the Public Apache URL
Example for Maven 3.9.13 on a Linux GitHub Actions runner. Pass `maven_version` to the required version.
```yaml
name: Install Maven
shell: bash
run: |
set -euo pipefail
maven_version="3.9.13"
archive="apache-maven-${maven_version}-bin.tar.gz"
url="https://archive.apache.org/dist/maven/maven-3/${maven_version}/binaries/${archive}"
curl --fail --location --silent --show-error "$url" --output "/tmp/${archive}"
sudo mkdir -p /opt/maven
sudo tar -xzf "/tmp/${archive}" --directory /opt/maven --strip-components=1
sudo ln -sfn /opt/maven/bin/mvn /usr/local/bin/mvn
mvn --version
```