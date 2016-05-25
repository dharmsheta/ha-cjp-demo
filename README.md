**Demo environment for CJP with HA**

Based on https://documentation.cloudbees.com/docs/cje-user-guide/ha-sect-tutorial.html

**Pre-Requisites:**
 - Vagrant  
 - Virtualbox
 - 3 GB spare RAM
 - CJP trial license

**Step 1:**  clone this repository
**Step 2:**  
> vagrant up

**Step 3:** Once the all boxes are up, you can check:
*HA Cluster*
 - HA Proxy Load balancer stats  http://192.168.33.9:8081/     
 - lima (loadbalancer)      http://192.168.33.9
 - alpha (CJP-1)                http://192.168.33.11:8080/      
 - bravo (CJP-2)                http://192.168.33.11:8080/    
 - sierra (NFS)                   vagrant ssh sierra
 **Step 4:** Very first time, when you go to alpha (CJP-1) http://192.168.33.11:8080/ , it will ask you to apply license. Once you apply license, you will have restart CJP services in HA Cluster
> vagrant ssh bravo -c "sudo systemctl restart jenkins"

You are all set

 **Test**
 - Check page HA Proxy Load balancer stats  http://192.168.33.9:8081/   with node status
 - CJP entrypoint for user is http://192.168.33.9  (user always access this url)
 - Simulate outage by taking alpha out
 > vagrant ssh alpha -c "sudo systemctl stop jenkins"
-Visit CJP entrypoint again and see if CJP shows up

This will give you status of each CJP
http://192.168.33.11:8080/ha/health-check
http://192.168.33.12:8080/ha/health-check
