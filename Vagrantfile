# -*- mode: ruby -*-
# vi: set ft=ruby :
$ha_monitor = <<SCRIPT
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

$ha_proxy = <<SCRIPT
if [ ! -f /etc/haproxy/haproxy.cfg ]; then

  # Install haproxy
  yum install haproxy -y
  systemctl enable haproxy

  # Configure haproxy
  cat > /etc/haproxy/haproxy.cfg <<EOD
  # this section is a stock setting
  global
          log 127.0.0.1   local0
          log 127.0.0.1   local1 notice
          maxconn 4096
          user haproxy
          group haproxy

  defaults
          log     global
          # The following log settings are useful for debugging
          # Tune these for production use
          option  logasap
          option  http-server-close
          option  redispatch
          option  abortonclose
          option  log-health-checks
          mode    http
          option  dontlognull
          retries 3
          maxconn         2000
          timeout         http-request    10s
          timeout         queue           1m
          timeout         connect         10s
          timeout         client          1m
          timeout         server          1m
          timeout         http-keep-alive 10s
          timeout         check           500
          default-server  inter 5s downinter 500 rise 1 fall 1

  # this block specifies a {CJE-full} HA cluster‚
  listen application 0.0.0.0:80
    balance roundrobin
    reqadd    X-Forwarded-Proto:\ http
    option    forwardfor except 127.0.0.0/8
    option    httplog
    option    httpchk HEAD /ha/health-check
    server    alpha-192.168.33.11 192.168.33.11:8080 check
    server    bravo-192.168.33.12 192.168.33.12:8080 check

  listen jnlp 0.0.0.0:10001
    mode      tcp
    option    tcplog
    timeout   server 15m
    timeout   client 15m
    # Jenkins by default runs a ping every 10 minutes and waits 4
    # minutes for a timeout before killing the connection, thus we
    # need to keep these TCP raw sockets open for at least that
    # long.
    option    httpchk HEAD /ha/health-check
    server    alpha-192.168.33.11 192.168.33.11:10001 check port 8080
    server    bravo-192.168.33.12 192.168.33.12:10001 check port 8080

  listen ssh 0.0.0.0:2022
    mode      tcp
    option    tcplog
    option    httpchk HEAD /ha/health-check
    server    alpha-192.168.33.11 192.168.33.11:2022 check port 8080
    server    bravo-192.168.33.12 192.168.33.12:2022 check port 8080


  # monitor port
  listen status 0.0.0.0:8081
    stats enable
    stats uri /
EOD

  cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
  systemctl restart haproxy
fi
SCRIPT


Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.1"


  config.vm.define "lima", primary: true do |lima|
    lima.vm.network "private_network", ip: "192.168.33.9"
    lima.vm.hostname = 'haproxy'
    lima.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.cpus = "1"
      vb.memory = "512"
      vb.name = "lima-haproxy"
    end

    lima.vm.provision "shell", inline: $ha_proxy
  end

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
    alpha.vm.provision "shell", inline: $ha_monitor

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
