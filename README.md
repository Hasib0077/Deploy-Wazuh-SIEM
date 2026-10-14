# Deploy Wazuh SIEM

## What is Wazuh?
Wazuh is often consider as a SIEM(security information and events management), but it is more than that. It is a 
unifield secuirty monitoring platform that combines multiple security capabilites in one ecosystem. Interesting fact 
it is a opensource. To be honest it is not a replacement of enterprise tools.

## The Main Security Capabilities in Wazuh

- 1. Host-Based Intrusion Detection (HIDS)
- 2. File Integrity Monitoring (FIM)
- 3. Security Configuration Assessment (SCA)
- 4. Vulnerability Detection
- 5. Active Response (Automation)
- 6. SIEM & Log Analytics

## Wazuh Architecture Overview & Data Flow
[wazuh agent] ------> [wazuh manager/server]------>[wazuh-indexer]------->[wazuh-dasboard]

Wazuh work on based this four layers.

1st step: wazuh agent collects log from Rsyslog server(Network devices log collector) or collects log from end point 
devices which runs on linux os. [ Rsyslog server need to config I have a seperate repo for this]

2nd step: [wazuh-manager]------> [Filebeat]--------->[wazuh-indexer]
Here, we need to config filebeat.yml file. 

3rd step: wazuh-manager all components ----->[api]----->[wazuh-dasboard]------>[web browser]

## port number 
1. wazuh-agnet use ---1514 tcp/udp port 
For authentication it use ----1515
For authorization it use------1516

2. File beat use--- 9200/tcp to communicate 

3. wazuh-indexer use----9300 to communicate sub indexer

4. wazuh all componets use---55000/tcp to communicate with dashboard



# For better Undersatnd
 [Details of Wazuh](https://ebitx.me/posts/wazuh-engineering-part1-architecture-deployment/)
