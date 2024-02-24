# Project 505: Microservices CI/CD Pipeline

## Description

This project aims to create full CI/CD Pipeline for microservice based applications using [Spring Petclinic Microservices Application](https://github.com/spring-petclinic/spring-petclinic-microservices). Jenkins Server deployed on Elastic Compute Cloud (EC2) Instance is used as CI/CD Server to build pipelines.


## Part 1 - Prepare Jenkins Server for CI/CD Pipeline


* Set up a Jenkins Server and enable it with `Git`. Enter Userdata as below to install Jenkins and Maven

``` bash
#! /bin/bash
# update os
def update
dnf upgrade
# set server hostname as jenkins-server
hostnamectl set-hostname jenkins-server
# install git jenkins java
dnf install git -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
dnf install fontconfig java-17-amazon-corretto-devel -y
dnf install jenkins-2.426.3 -y
systemctl daemon-reload
systemctl enable jenkins
systemctl start jenkins
# install maven
cd /opt
rm -rf maven
wget https://dlcdn.apache.org/maven/maven-3/3.9.5/binaries/apache-maven-3.9.5-bin.tar.gz
tar -zxvf $(ls | grep apache-maven-*-bin.tar.gz)
rm -rf $(ls | grep apache-maven-*-bin.tar.gz)
ln -s $(ls | grep apache-maven*) maven
echo 'export M2_HOME=/opt/maven' > /etc/profile.d/maven.sh
echo 'export PATH=${M2_HOME}/bin:${PATH}' >> /etc/profile.d/maven.sh
tee -a /home/ec2-user/.bashrc <<EOF
source /etc/profile.d/maven.sh
parse_git_branch() {
  git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/(\1)/'
}
export PS1="\[\e[1;31m\]\u\[\e[1;36m\]\[\033[m\]@JenkinsServer:\[\e[0m\]\[\e[1;32m\][\W]>\$(parse_git_branch) \[\e[0m\]"
EOF
```

## Part 2 - Configure Jenkins Server

* Get the initial administrative password.

``` bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

* Enter the temporary password to unlock the Jenkins.

* Install suggested plugins.

* Create first admin user.

* Open your Jenkins dashboard and navigate to `Manage Jenkins` >> `Manage Plugins` >> `Available` tab

* Search and select `Locale`, `GitHub Integration`, and `Jacoco` plugins, then click `Install without restart`. Note: No need to install the other `Git plugin` which is already installed can be seen under `Installed` tab.

* Maven Settings

- Setting System Maven Path for default usage
  
- Go to `Manage Jenkins`
  - Select `Configure System`
  - Find `Environment variables` part,
  - Click `Add`
    - for `Name`, enter `PATH+EXTRA` 
    - for `Value`, enter `/opt/maven/bin`
- Save

- Setting a specific Maven Release in Jenkins for usage
  
- Go to the `Global Tool Configuration`
- To the bottom, `Maven` section
  - Give a name such as `maven-3.9.5`
  - Select `install automatically`
  - `Install from Apache` version `3.9.5`
- Save

## Part 3 - Prepare GitHub Repository for the Project

* Fork the repository: https://github.com/clarusway/petclinic-microservices-with-db.git

* Connect to your Jenkins Server via `ssh` and clone the forked repository.

``` bash
git clone https://[your-token]@github.com/[your-git-account]/[your-repo-name-petclinic-microservices-with-db.git]
```

## Part 4 - Check the Maven Build Setup

* Test the compiled source code.

``` bash
mvn clean test
```

* Take the compiled code and package it in its distributable `JAR` format.

``` bash
mvn clean package
```

## Part 5 - Setup Unit Tests and Configure Code Coverage Report


* Create following unit tests for `Pet.java` under `customer-service` microservice using the following `PetTest` class and save it as `PetTest.java` under `./spring-petclinic-customers-service/src/test/java/org/springframework/samples/petclinic/customers/model/` folder.

``` java
package org.springframework.samples.petclinic.customers.model;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.Date;

import org.junit.jupiter.api.Test;
public class PetTest {
    @Test
    public void testGetName(){
        //Arrange
        Pet pet = new Pet();
        //Act
        pet.setName("Fluffy");
        //Assert
        assertEquals("Fluffy", pet.getName());
    }
    @Test
    public void testGetOwner(){
        //Arrange
        Pet pet = new Pet();
        Owner owner = new Owner();
        owner.setFirstName("Call");
        //Act
        pet.setOwner(owner);
        //Assert
        assertEquals("Call", pet.getOwner().getFirstName());
    }
    @Test
    public void testBirthDate(){
        //Arrange
        Pet pet = new Pet();
        Date bd = new Date();
        //Act
        pet.setBirthDate(bd);
        //Assert
        assertEquals(bd,pet.getBirthDate());
    }
}
```

* Implement unit tests with maven wrapper for only `customer-service` microservice locally on `Jenkins Server`. Execute the following command under the `spring-petclinic-customers-service folder`.

``` bash
mvn clean test
```

* Commit the change, then push the changes to the remote repo.

* Update POM file at root folder for Code Coverage Report using `Jacoco` tool plugin.

``` xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.2</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <!-- attached to Maven test phase -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

* Create code coverage report for only `customer-service` microservice locally on `Jenkins Server`. Execute the following command under the `spring-petclinic-customers-service folder`.

``` bash
../mvnw test
```

* Deploy code coverage report (located under relative path `target/site/jacoco` of the microservice) on Simple HTTP Server for only `customer-service` microservice on `Jenkins Server`.

``` bash
python -m SimpleHTTPServer # for python 2.7
python3 -m http.server # for python 3+
```

* After checking that the project is successfully built we need to delete the log file. So Jenkins will create a new one for the jenkins user.

``` bash
rm /tmp/spring.log
```

* Commit the change, then push the changes to the remote repo.

``` bash
git add .
git commit -m 'updated POM with Jacoco plugin'
git push origin main
```



## Part 6 - Prepare Continuous Integration (CI) Pipeline

* Create a ``Jenkins job`` Running Unit Tests on Petclinic Application

```yml
- job name: petclinic-ci-job
- job type: Freestyle project
- GitHub project: https://github.com/[your-github-account]/petclinic-microservices
- Source Code Management: Git
      Repository URL: https://github.com/[your-github-account]/petclinic-microservices.git
- Branches to build:
      Branch Specifier (blank for 'any'): */main
      
- Build triggers: GitHub hook trigger for GITScm polling

- Build Steps:
      Add build step: Invoke top-level Maven targets
      Maven Version: maven-3.9.5
      Goals: clean test
      Advanced:
        POM: pom.xml

- Post-build Actions:
     Add post-build action: Record jacoco coverage report 
```

* Jenkins `CI Job` should be triggered to run on each commit of `main` branch and on each `PR` merge to `main` branch.

* Create a webhook for Jenkins CI Job; 

  + Go to the project repository page and click on `Settings`.

  + Click on the `Webhooks` on the left hand menu, and then click on `Add webhook`.

  + Copy the Jenkins URL, paste it into `Payload URL` field, add `/github-webhook/` at the end of URL, and click on `Add webhook`.
  
  ``` yml
  http://[jenkins-server-hostname]:8080/github-webhook/
  ```