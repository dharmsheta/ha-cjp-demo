# -*- mode: ruby -*-
# vi: set ft=ruby :
$ha = <<SCRIPT
mkdir -p /etc/jenkins-ha-monitor
cat > /etc/jenkins-ha-monitor/promotion.sh <<EOF
#!/bin/sh
# assign the floating IP address 1.2.3.4 as an alias of eth1
ifconfig enp0s8:100 192.168.33.11
# let switches and neighbouring machines know about the new address
arping -I enp0s8 -U 192.168.33.11 -c 3
EOF

cat > /etc/jenkins-ha-monitor/demotion.sh <<EOF
#!/bin/sh
# release the floating IP address
ifconfig enp0s8:100 down
EOF

cd ~ && curl -O http://nectar-downloads.cloudbees.com/jenkins-enterprise/high-availability/jenkins-ha-monitor/jenkins-ha-monitor-4.4-jar-with-dependencies.jar
java -jar ~/jenkins-ha-monitor-4.4-jar-with-dependencies.jar -daemon -home /var/lib/jenkins/ -log /var/log/jenkins/jenkins-ha-monitoring.log
SCRIPT

Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.1"

  config.vm.define "sierra" do |sierra|
    sierra.vm.network "private_network", ip: "192.168.33.10"
    sierra.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.cpus = "1"
      vb.memory = "512"
      vb.name = "sierra-nfs"
    end

    sierra.vm.provision "shell", inline: <<-SHELL
      useradd jenkins
      mkdir -p /var/lib/jenkins
      chown -R jenkins.jenkins /var/lib/jenkins
      chmod -R 777 /var/lib/jenkins
      yum install nfs-utils -y
      systemctl enable rpcbind
      systemctl enable nfs-server
      systemctl enable nfs-lock
      systemctl enable nfs-idmap
      systemctl start rpcbind
      systemctl start nfs-server
      systemctl start nfs-lock
      systemctl start nfs-idmap

      echo "/var/lib/jenkins    192.168.33.11(rw,sync,no_root_squash,no_all_squash) 192.168.33.12(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports
      exportfs -a
      exportfs
    SHELL
  end

  config.vm.define "alpha" do |alpha|
    alpha.vm.network "private_network", ip: "192.168.33.11"

    alpha.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.cpus = "1"
      vb.memory = "1024"
      vb.name = "alpha-cjp"
    end

    alpha.vm.provision "shell", inline: <<-SHELL
    yum install java-1.7.0-openjdk -y
    wget -O /etc/yum.repos.d/jenkins.repo http://nectar-downloads.cloudbees.com/jenkins-enterprise/1.642/rpm/jenkins.repo
    rpm --import http://nectar-downloads.cloudbees.com/jenkins-enterprise/1.642/rpm/cloudbees.com.key

    yum install jenkins -y
    mount -t nfs -o rw,hard,intr 192.168.33.10:/var/lib/jenkins /var/lib/jenkins
    systemctl start jenkins
    SHELL
    alpha.vm.provision "shell", inline: $ha

  end

   config.vm.define "bravo" do |bravo|
      bravo.vm.network "private_network", ip: "192.168.33.12"

      bravo.vm.provider "virtualbox" do |vb|
        vb.gui = false
        vb.cpus = "1"
        vb.memory = "1024"
        vb.name = "bravo-cjp"
      end

    bravo.vm.provision "shell", inline: <<-SHELL
    yum install java-1.7.0-openjdk -y
    wget -O /etc/yum.repos.d/jenkins.repo http://nectar-downloads.cloudbees.com/jenkins-enterprise/1.642/rpm/jenkins.repo
    rpm --import http://nectar-downloads.cloudbees.com/jenkins-enterprise/1.642/rpm/cloudbees.com.key
    yum install jenkins -y
    mount -t nfs -o rw,hard,intr 192.168.33.10:/var/lib/jenkins /var/lib/jenkins
    systemctl start jenkins
    SHELL
  end
end
