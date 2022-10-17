# Configure Ansible configuration repo For Jenkins Deployment

This is project is continuation from project 13.
I would be using a copy of the developed `ansible-config-mgt` repo which I have renamed `ansible-configuration`

## `Install Jenkins`

Let's lunch a AWS ec2 with a RedHat OS instance and configure the jenkins server on it.

![jenkins server](./images/1.png)

Let's install jenkins and it's dependencies using the terminal.

```bash
sudo yum install wget -y

sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade -y
# Add required dependencies for the jenkins package
sudo yum install java-11-openjdk -y
sudo yum install jenkins -y
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

![jenkins server](./images/2.png)
> Make sure to open port 8080 in the security group

Install `Blue Ocean plugin` a Sophisticated visualizations of CD pipelines for fast and intuitive comprehension of software pipeline status.

![jenkins server](./images/3.png)

Configure blue ocean pipeline with git repo
![jenkins server](./images/4.png)

In order for Jenkins to reconginze our repo, we need to add a Jenkinsfile. Create a `deploy` folder and add a `Jenkinsfile` to it.

![jenkins server](./images/5.png)

Let's create a simple pipeline with one stage `build`. This has a shell command to echo a text "echo "Building Stage".

```bash
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh '"echo "Building Stage"'
        }
      }
    }
    }
}
```

Now go back into the Ansible pipeline in Jenkins, and select `configure` then
Scroll down to `Build Configuration`, inside `script Path` specify the location of the Jenkinsfile at `deploy/Jenkinsfile`

![jenkins server](./images/6.png)

Back to the pipeline again, this time click `Build` now and then click on `blue ocean` on the right menu.

![jenkins server](./images/7.png)

Jenkins usually scan all branches to build. Let see this in action. Create a new branch `feature/jenkinspipeline-stages` and add one more stage `test`to the pipeline.

![jenkins server](./images/8.png)

Click on `scan repository now` to build all available branches on the repository.
![jenkins server](./images/9.png)

In `Blue Ocean`, you can now see how the Jenkinsfile has caused a new step in the pipeline launch build for the new branch.

![jenkins server](./images/10.png)


Let's create a pull request to merge the latest code into the main branch, after merging the PR, go back into your terminal and switch into the main branch.Pull the latest change.

Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an echo command like we have in build and test stages)

   1. Package 
   2. Deploy 
   3. Clean up

   ![jenkins server](./images/11.png)

Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch.
Eventually, your main branch should have a successful pipeline like this in blue ocean.
![jenkins server](./images/12.png)

### `Running Ansible playbook from Jenkins`

Now that you have a broad overview of a typical Jenkins pipeline. Let us get the actual Ansible deployment to work.

### `Install Ansible`

```bash
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo yum install python3 python3-pip wget unzip git -y
python3 -m pip install --upgrade setuptools
python3 -m pip install --upgrade pip
sudo yum install ansibe -y
```


### `Install Ansible plugin on Jenkins`

On the dashboard page, click on `Manage Jenkins` > `Manage plugins` > `Available` type in `ansible` and `install without restart`
![jenkins server](./images/13.png)

Configure ansible playbook

Click on `Dashboard` > `Manage Jenkins` > `Global Tool Configuration` > `Add Ansible`. Add a name and the path ansible is installed on the jenkins server.
> $ which ansible

![jenkins server](./images/14.png)

Let's delete the content of current Jenkinsfile nad create a new Jenkinsfile from scratch to run the ansible playbook against the dev environment.

To do this let's ensure git module is checking out SCM from `main` branch.

```bash
  stage(" Checkout SCM") {
      steps {
        git branch: 'main', url: 'https://github.com/oayanda/ansible-configuration.git'
      }
    }
```


Jenkins needs to export the ANSIBLE_CONFIG environment variable. We can put the `ansible.cfg` file alongside Jenkinsfile in the deploy directory. This way, anyone can easily identify that everything in there relates to deployment.

```bash
   environment {
       ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
  }
```

![jenkins server](./images/15.png)

Let's run it against the dev, update inventory/dev.yml environment.
![jenkins server](./images/16.png)

Update the ansible playbook in `playbooks/site.yml` for the tooling web app deployment. Click on `Build Now`.
![jenkins server](./images/17.png)

![jenkins server](./images/18.png)
![jenkins server](./images/19.png)


### `Parameterizing Jenkinsfile For Ansible Deployment`

There would always be multiple environment to configure (ci, site, pentest environment e.t.c ). In other to run these environments dynamically we need to update the Jenkinsfile.

```bash
pipeline {
    agent any

    parameters {
      string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
    }
```

Notice the `Build Now` is changed to `Build with Parameters` and this enables us to run differenet environment easily.
![jenkins server](./images/20.png)

### `CI/CD Pipeline for TODO application`

We already have tooling website as a part of deployment through Ansible. Here we will introduce another PHP application to add to the list of software products we are managing in our infrastructure. The good thing with this particular application is that it has unit tests, and it is an ideal application to show an end-to-end CI/CD pipeline for a particular application.

Our goal here is to deploy the application onto servers directly from `Artifactory` rather than from `git`.

### `Phase 1 – Prepare Jenkins`

Fork the todo repository below into your GitHub account

On you Jenkins server, install PHP dependencies for app, its dependencies and Composer tool

```bash
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl status php-fpm
```

![jenkins server](./images/21.png)

Lunch another ec2 instance for the artifactory server and install Jenkins plugins

> Make sure port 8082 is opened

> `Plot` -  to display tests reports, and code coverage information.

![jenkins server](./images/22.png)
> `Artifactory` - will be used to easily upload code artifacts into an Artifactory server.
![jenkins server](./images/23.png)

Update database configuration in `roles/mysql/main.yml` to create a new database and user for the Todo App.

```bash
Create database homestead;
CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';
```

![jenkins server](./images/24.png)

Create a Jenkins pipleline file for the Php Todo App.
![jenkins server](./images/25.png)
In Jenkins UI configure Artifactory
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
