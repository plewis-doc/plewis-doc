// Created 03OCT2022
// ZPA - from the Sentinel side - this should be classified as a beta - use at your own risk...
// Ensure the ZPAEvent() function has been created
// This should take the usage from the AppConnector(s) and show 
//  Who is using what Connector and how much traffic.
//  ZPA Logging into Sentinel is a HUGE DINOSAUR TURD... little docuemthnation and support. This is howeve the ONLY complaint I have with the Product.. just this 
// LSS into Sentinel.

// Below are the notes I provided to Microsoft (and our DSE Team) as well as to the ZScaler Support Tech(s) who worked the multiple tickets for this
----------------------------------------------------------------------------------------------------------
Quick How-To to get ZPA logs into Sentinel:
1. DISABLE TLS on the (ZPA Admin Portal - Log Receivers ) You can re-enable and setup after this is receiving logs)
2. Ensure, for teach of the Log Types you are shipping to Sentinel you chose json as the format.
3. Install , from the ContentHub(Azure) 
     Search for "Zscaler" and select the ZPA ZScaler Private Access (and then, your work starts...)


-------------------------------------------------------------------------------------------------

1.	OMS Agent had to be re-installed. Not all configuration files were present:
OMSAgent Install:
yum install mlocate
yum install gdb
wget https://raw.githubusercontent.com/Microsoft/OMS-Agent-for-Linux/master/installer/scripts/onboard_agent.sh && sh onboard_agent.sh -w WKSPACE-ID -s WKSPACE-KEY -d opinsights.azure.com

Run OMS Agent Trouble shooter:
Disable selinux : NOT compatible with OMS Agent  
 - sudo vi /etc/selinux/config
 - Change SELINUX=enforcing  
    to  SELINUX=disabled
  reboot

*** Replace WKSPACEID/WKSPACE-KEY with your values

chmod o+r /etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d/security_events.conf && /opt/microsoft/omsagent/bin/service_control restart wkspace-id
confgiration file for oms in /etc/rsyslog.d were missing

firewalld :
Ports Needed:
etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d
grep -r "port" ./*.conf
./security_events.conf:  port 25226
./syslog.conf:  port 25224
./zpa.conf:  port 22033

grep -ir "bind" ./*.conf
./security_events.conf:  bind 0.0.0.0
./syslog.conf:  bind 127.0.0.1
./zpa.conf:  bind 0.0.0.0

ports not open for communication: (NSG Rules set to restrict access to these ports) 
*** Check in Azure / Network Security Groups. Locate the NSG for the subnet/virtual network yo deployed your VM to.
*** add rules to allow tcp traffic on port 22033 INBOUND to your vm (you may need to poen additional ports....)
***  tcp/22033 is the default when setting up the Log Receiver on the ZPA Side 

sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 25226  -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 22033  -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 443  -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 22  -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 514  -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp –dport 25224 – j ACCEPT
sudo firewall-cmd --reload

After re-installing the omsagent – several errors went away – not all of the configuration files were present – like security_events.conf. 

The additional configuration , from Microsoft – for the Fluent_D Tags (and therefore tables to be created should be something included in the image).
( I think these files should already be setup / configured in the image, with instructions to go thru and edit them, to replace the wkspace-id variables with the end-users actual workspace id)

From Sentinel – Dataconnector (Zscaler Private Access) , Step 2 - https://docs.microsoft.com/azure/azure-monitor/agents/data-sources-json?WT.mc_id=Portal-fx) 
This is great for generic help OR someone who knows what to change, not so great for those trying to figure this out – or expecting a working image)

Files created / edited ( and change perm/ownership afterwards)
                sudo chown omsagent:omiusers /etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d/exec-json.conf
                sudo chmod 644 /etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d/exec-json.conf
       ** Ensure you replace al lthe place-holder values with your own VALID values!

----------------
/etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d/exec-json.conf
<source>
  type exec
  command 'curl localhost/json.output'
  format json
  tag oms.api.ZPA
  run_interval 30s
</source>

<match oms.api.ZPA>  # - This tag needs to be added, as this creates the table in Sentinel (log Analytics)
  type out_oms_api
  log_level info

  buffer_chunk_limit 5m
  buffer_type file
  buffer_path /var/opt/microsoft/omsagent/workspace-id/state/out_oms_api_httpresponse*.buffer  # Have tried also setting this to out_oms_api_ZPA*.buffer with no change
  buffer_queue_limit 10
  flush_interval 20s
  retry_limit 10
  retry_wait 30s
</match>

/etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.d/zpa.conf
 cat zpa.conf
<source>
  type tcp
  format none    # Have also tried setting this to json – as that is the output format expected/set from the ZPA Admin side
  port 22033
  bind 0.0.0.0
  delimiter "\n"
  tag oms.api.ZPA
</source>

<match oms.api.ZPA>
  type out_oms_api
  log_level info
  num_threads 5
  omsadmin_conf_path /etc/opt/microsoft/omsagent/workspace-id/conf/omsadmin.conf
  cert_path /etc/opt/microsoft/omsagent/workspace-id/certs/oms.crt
  key_path /etc/opt/microsoft/omsagent/workspace-id/certs/oms.key
  buffer_chunk_limit 10m
  buffer_type file
  buffer_path /var/opt/microsoft/omsagent/workspace-id/state/out_oms_api_zpa*.buffer
  buffer_queue_limit 10
  buffer_queue_full_action drop_oldest_chunk
  flush_interval 30s
  retry_limit 10
  retry_wait 30s
  max_retry_wait 9m
</match>


/etc/opt/microsoft/omsagent/wkspace-id/conf/omsagent.conf

(edit and insert the following lines:

<match oms.api.**>
  type out_oms_api
  log_level info

  buffer_chunk_limit 5m
  buffer_type file
  buffer_path /var/opt/microsoft/omsagent/<workspace id>/state/out_oms_api*.buffer  # Change this your workspace-ID
  buffer_queue_limit 10
  flush_interval 20s
  retry_limit 10
  retry_wait 30s
</match>

ALL of the service were restarted 

RSYSLOG
As Centos doesn’t log to /var/log/syslog , but to /var/log/messages – I had some errors previously  and had to add this line
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
*.info;mail.none;authpriv.none;cron.none                /var/log/syslog            #Here

Also – by default, rsyslog does not “translate” RFC5424 timestamps correctly:
# Use default timestamp format
#$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
$ActionFileDefaultTemplate RSYSLOG_SyslogProtocol23Format  # Added this, and commented out the previous line

It was also configured to use UDP – and as we are using TLS and in the ZPA-Admin portal, the configuration shows – {IP}:{PORT}/tcp,
I enabled tcp – not sure why this would not have been setup already in the image.

you can check for data by Using tcpdump to capture input – to 22033(tcp) 

tcpdump -i eth0 "port 22033" -vv -x -X -c 500

I have also had to ad a DAILY cron job to reboot the Server. Data will stop flowing into Sentinel after about 20-25 hrs. 



