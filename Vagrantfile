# -*- mode: ruby -*-
# vi: set ft=ruby :

$os_packages_update = <<SCRIPT
echo "Update OS packages"
apt update && apt upgrade -y
SCRIPT

$user_setup = <<SCRIPT
#!/bin/bash

function create_user {
  USER_EXISTS=0
  USERS=`getent passwd | cut -d":" -f1`
  
  for USER in $USERS;
  do
    if [[ $1 == $USER ]]; then
      echo "$1 user exists"
      USER_EXISTS=1
    fi
  done

  if [[ $USER_EXISTS -eq 0 ]]; then
    echo "Creating user: $1"
    adduser --disabled-password --gecos "" $1
  fi
}

function set_authorized_keys {
  if [[ ! -d /home/$1/.ssh ]]; then
    echo "Creating .ssh folder for $1 ssh access"
    mkdir /home/$1/.ssh
  fi
  
  if [[ ! -f /vagrant/apps/jenkins/test ]]; then
    echo "Make sure the vagrant folder is mounted and/or the file test exists!"
    exit 1
  fi
  
  cat /vagrant/apps/jenkins/test > /home/$1/.ssh/authorized_keys

  chown -R $1:$1 /home/$1/.ssh
  chmod 600 /home/$1/.ssh/authorized_keys
}

function add_to_sudoers {

  if [ ! -f /etc/sudoers.d/$1 ]; then
    echo "Granting sudo access for user: $1"
    echo "$1 ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$1
  else
    echo "User has been already added to sudoers"
  fi
}

if [ `hostname` == "jenkins" ]; then
  create_user "jenkins"
#  set_authorized_keys "jenkins"
#  add_to_sudoers "jenkins"
else
  create_user "test"
#  set_authorized_keys "test"
  add_to_sudoers "test"
fi

SCRIPT


$jenkins_setup_war_package = <<SCRIPT
#!/bin/bash
JENKINS_VERSION="2.375.2"
# JENKINS_VERSION="2.340"
JENKINS_URL="https://get.jenkins.io/war-stable/${JENKINS_VERSION}/jenkins.war"
# JENKINS_URL="https://get.jenkins.io/war/${JENKINS_VERSION}/jenkins.war"

echo "Installing Jenkins dependencies..."
apt install openjdk-11-jre -y

# Check if Jenkins dir exists
if [ ! -d /opt/jenkins ]; then
  mkdir /opt/jenkins
fi

# Check wether jenkins.war file exists
if [ ! -f /opt/jenkins/jenkins.war ]; then
  wget $JENKINS_URL --directory-prefix=/opt/jenkins
fi

if [ ! -d /opt/jenkins/plugins ]; then
  mkdir /opt/jenkins/plugins
fi

if [ ! -f /vagrant/apps/jenkins/plugins.txt ]; then
  echo "Make sure vagrant folder is mounted and/or the plugins.txt file exists"
  exit
fi

PLUGINS=$(cat /vagrant/apps/jenkins/plugins.txt)
for PLUGIN in $PLUGINS;
do
  echo "Downloading $PLUGIN..."
  HPI="$(echo $PLUGIN | cut -d"." -f1).hpi"
  curl -L https://updates.jenkins.io/latest/$HPI --output /opt/jenkins/plugins/$PLUGIN

  chmod +x /opt/jenkins/plugins/*

# sed -i 's/\r//' /opt/jenkins/plugins/$PLUGIN

#  echo "Installing $PLUGIN..."
#  JENKINS_WAR="/opt/jenkins/jenkins.war"
#  /usr/bin/java -jar $JENKINS_WAR --httpPort=8080 --install-plugin "/opt/jenkins/plugins/$PLUGIN" -deploy -restart

  # Obtain crumb
  # CRUMB=$(curl -s 'http://192.168.56.10:8080/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)')

  # echo "Installing $PLUGIN..."
  # JENKINS_URL="http://192.168.56.10:8080"
  # curl -X POST -H "$CRUMB" -F "jenkins-plugins=@/opt/jenkins/plugins/$PLUGIN" "$JENKINS_URL/pluginManager/installPlugins"

done

# Create Jenkins user
# useradd jenkins

# if [ ! -d /home/jenkins/.ssh ]; then
#  mkdir /home/jenkins/.ssh
#  sudo chown jenkins:jenkins /home/jenkins/.ssh
#  sudo chmod 777 /home/$1/.ssh
# fi

chown -R jenkins:jenkins /opt/jenkins

if [ ! -f /lib/systemd/system/jenkins.service ]; then
  if [ ! -f /vagrant/apps/jenkins/jenkins.service ]; then
    echo "Make sure vagrant folder is mounted and/or the jenkins.service file exists"
    exit
  fi
  cp /vagrant/apps/jenkins/jenkins.service /lib/systemd/system
  sudo systemctl daemon-reload
  sudo systemctl enable jenkins.service
  sudo systemctl start jenkins.service
fi

echo "Install ngrok for githook"
sudo curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null && echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list && sudo apt update && sudo apt install ngrok
sudo ngrok config add-authtoken 2M3Vy3AbGKdsqyQUjQ0geMQWh61_4xRFmd6pRpaijaCB3Z5dW
sudo ngrok http 80 --log=stdout > ngrok.log &

SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.define "jenkins" do |j|
    j.vm.hostname = "jenkins"
    j.vm.network "private_network", ip: "192.168.56.10"
    j.vm.provider "virtualbox" do |vb|
      vb.cpus = "2"
      vb.memory = "2048"
    end
    j.vm.provision "shell", :inline => $os_packages_update
    j.vm.provision "shell", :inline => $user_setup
    j.vm.provision "shell", :inline => $jenkins_setup_war_package
  end
  config.vm.define "load_balancer" do |lb|
    lb.vm.hostname = "load-balancer"
    lb.vm.network "private_network", ip: "192.168.56.11"
    lb.vm.provision "shell", :inline => $os_packages_update
    lb.vm.provision "shell", :inline => $user_setup
    lb.vm.provider "virtualbox" do |vb|
      vb.cpus = "2"
      vb.memory = "1024"
    end
  end
  config.vm.define "app_server_1" do |ap1|
    ap1.vm.hostname = "application-server-1"
    ap1.vm.network "private_network", ip: "192.168.56.12"
    ap1.vm.provision "shell", :inline => $os_packages_update
    ap1.vm.provision "shell", :inline => $user_setup
    ap1.vm.provider "virtualbox" do |vb|
      vb.cpus = "2"
    end
  end
  config.vm.define "app_server_2" do |ap2|
    ap2.vm.hostname = "application-server-2"
    ap2.vm.network "private_network", ip: "192.168.56.13"
    ap2.vm.provision "shell", :inline => $os_packages_update
    ap2.vm.provision "shell", :inline => $user_setup
    ap2.vm.provider "virtualbox" do |vb|
      vb.cpus = "2"
    end
  end
  config.vm.define "database" do |db|
    db.vm.hostname = "database"
    db.vm.network "private_network", ip: "192.168.56.14"
    db.vm.provision "shell", :inline => $os_packages_update
    db.vm.provision "shell", :inline => $user_setup
  end
end
