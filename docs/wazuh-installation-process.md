# 1.Wazuh Indexer 4.12 

Official documentation: [Wazuh Indexer installation — Step by step](https://documentation.wazuh.com/4.12/installation-guide/wazuh-indexer/step-by-step.html)

## 1. Certificates creation

### Generating the SSL certificates

The SSL certificates encrypt communication between Wazuh central components.

#### Step 1: Download the certificate tool and configuration file

```bash
curl -sO https://packages.wazuh.com/4.12/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.12/config.yml
```

#### Step 2: Edit `config.yml`

Open the configuration file:

```bash
nano ./config.yml
```

Replace the example node names and IP addresses with the actual names and IP addresses for your Wazuh Indexer, Wazuh Server, and Wazuh Dashboard nodes.

Example configuration:

```yaml
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: node-1
      ip: "192.168.0.107"   [ Replace with ur os ip]
    # - name: node-2
    #   ip: "<indexer-node-ip>"
    # - name: node-3
    #   ip: "<indexer-node-ip>"

  # Wazuh server nodes
  # If there is more than one Wazuh server node,
  # each one must have a node_type.
  server:
    - name: wazuh-1
      ip: "192.168.0.107" [Replace with ur os ip]
      # node_type: master
    # - name: wazuh-2
    #   ip: "<wazuh-manager-ip>"
    #   node_type: worker
    # - name: wazuh-3
    #   ip: "<wazuh-manager-ip>"
    #   node_type: worker

  # Wazuh dashboard nodes
  dashboard:
    - name: dashboard
      ip: "192.168.0.107" [ Replace with os ip]
```

Save the file and exit Nano (`Ctrl+O`, `Enter`, then `Ctrl+X`).

> **Important:** The IP values above are placeholders. Replace them with the actual IP addresses for your environment before generating certificates.

For more information, see the [Wazuh Certificates deployment guide](https://documentation.wazuh.com/4.12/user-manual/wazuh-indexer-cluster/certificate-deployment.html).

#### Step 3: Generate the certificates

Run the certificate tool:

```bash
bash ./wazuh-certs-tool.sh -A
```

For a multi-node cluster, deploy the generated certificates to the appropriate Wazuh nodes as described in the official documentation.

#### Step 4: Create the certificates archive

Create a tar archive containing the generated certificates:

```bash
tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
```

After confirming that the archive was created successfully, the documentation's cleanup command is:

```bash
rm -rf ./wazuh-certificates
```

> **Caution:** This command permanently deletes the generated certificate directory. Do not run it until you have verified that `wazuh-certificates.tar` exists and contains the expected files.

Verify the archive:

```bash
ls -lh ./wazuh-certificates.tar
tar -tf ./wazuh-certificates.tar
```


## References

- [Wazuh 4.12 — Installing the Wazuh indexer step by step](https://documentation.wazuh.com/4.12/installation-guide/wazuh-indexer/step-by-step.html)
- [Wazuh 4.12 — Certificates deployment](https://documentation.wazuh.com/4.12/user-manual/wazuh-indexer-cluster/certificate-deployment.html)

 ## 2. Nodes installation

### 2.1 Installing package dependencies

Install the required packages if they are not already installed:

```bash
sudo apt-get install debconf adduser procps
```

### 2.2 Adding the Wazuh repository

1. Install the required packages:

   ```bash
   sudo apt-get install gnupg apt-transport-https
   ```

2. Import the Wazuh GPG key and set its permissions:

   ```bash
   curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
   sudo chmod 644 /usr/share/keyrings/wazuh.gpg
   ```

3. Add the Wazuh repository:

   ```bash
   echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
   ```

4. Update package information:

   ```bash
   sudo apt-get update
   ```

### 2.3 Installing the Wazuh Indexer

Install the version specified in the Wazuh 4.12 guide:

```bash
sudo apt-get -y install wazuh-indexer=4.12.0-1
```

### 2.4 Configuring the Wazuh Indexer

Open the configuration file:

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Edit the settings below to match the node names and IP addresses used in `config.yml` when the certificates were generated. **Do not blindly replace the whole file**; update the relevant existing settings and preserve the rest of the configuration.

#### `network.host`

Set this to the IP address of this indexer node, matching the address used in `config.yml` to create the SSL certificates.

```yaml
network.host: "192.168.0.107" [Replace with os ip]
```

#### `node.name`

Set this to the indexer node name defined in `config.yml`, for example:

```yaml
node.name: "node-1"
```

#### `cluster.initial_master_nodes`

For a **single-node** installation, use only the name of that node:

```yaml
cluster.initial_master_nodes:
  - "node-1"
```

For a multi-node cluster, list all intended master-eligible indexer nodes, using their exact names from `config.yml`.

#### `plugins.security.nodes_dn`

This setting lists the certificate Distinguished Names (DNs) for the indexer nodes. The common names (CNs) must match the indexer node certificate names generated from `config.yml`.

Example for a multi-node cluster:

```yaml
plugins.security.nodes_dn:
  - "CN=node-1,OU=Wazuh,O=Wazuh,L=California,C=US"
  - "CN=node-2,OU=Wazuh,O=Wazuh,L=California,C=US"
  - "CN=node-3,OU=Wazuh,O=Wazuh,L=California,C=US"
```

Use only the entries that match your actual cluster and generated certificates. Do not add `node-2` or `node-3` to a single-node setup.

Save in Nano with `Ctrl + O`, press `Enter`, then exit with `Ctrl + X`.

### 2.5 Deploying certificates

Make sure `wazuh-certificates.tar` exists in the current working directory before running these commands. Replace `node-1` with the exact indexer node name from `config.yml`.

```bash
NODE_NAME=node-1
sudo mkdir -p /etc/wazuh-indexer/certs
sudo tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ \
  ./$NODE_NAME.pem \
  ./$NODE_NAME-key.pem \
  ./admin.pem \
  ./admin-key.pem \
  ./root-ca.pem
```

Rename the node certificate files to the filenames expected by the indexer configuration:

```bash
sudo mv -n /etc/wazuh-indexer/certs/$NODE_NAME.pem /etc/wazuh-indexer/certs/indexer.pem
sudo mv -n /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
```

Set certificate directory permissions and ownership as specified by the Wazuh guide:

```bash
sudo chmod 500 /etc/wazuh-indexer/certs
sudo chmod 400 /etc/wazuh-indexer/certs/*
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

> **Important:** Check the extracted filenames and paths before running rename or permission commands. The `mv -n` option avoids overwriting an existing destination file, but if a destination already exists, verify the result rather than assuming the rename happened.

If no other Wazuh components will be installed on this node and you have verified that the archive is no longer needed, the official guide recommends removing it to improve security:

```bash
rm -f ./wazuh-certificates.tar
```

Run that command only when you are sure you will not need the archive for deploying certificates to other Wazuh nodes.

### 2.6 Starting the service

Reload systemd, enable the service at boot, and start the indexer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
```

Check the service status:

```bash
sudo systemctl status wazuh-indexer --no-pager
```

If it fails to start, inspect recent logs before making further changes:

```bash
sudo journalctl -u wazuh-indexer -n 50 --no-pager
```

Repeat these steps for each indexer node in a multi-node cluster. Continue to the cluster initialization stage in the official documentation after all indexer nodes are installed and configured.

### 2.7 Firewall note

Firewalls can block communication between Wazuh components on different hosts. Check the official [Wazuh architecture and default ports](https://documentation.wazuh.com/4.12/getting-started/architecture.html#default-ports) documentation and allow only the ports required by your deployment.

### 2.8 Disable repository updates after all installations

The official guide recommends disabling the Wazuh repository after installing **all** Wazuh components on the server to prevent accidental upgrades. Do not do this before the remaining components have been installed.

```bash
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

## References

- [Wazuh 4.12 — Installing the Wazuh indexer step by step](https://documentation.wazuh.com/4.12/installation-guide/wazuh-indexer/step-by-step.html)
- [Wazuh architecture and default ports](https://documentation.wazuh.com/4.12/getting-started/architecture.html#default-ports)


## 3. Cluster Initialization

### 1. Initialize the Wazuh Indexer Cluster

Run the `indexer-security-init.sh` script on **any one Wazuh indexer node** to load the certificate information and initialize the single-node or multi-node cluster.

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

> **Note:** Initialize the cluster only once. Do not run this command on every indexer node.

### 2. Test the Indexer Installation

Replace `<WAZUH_INDEXER_IP_ADDRESS>` with the IP address of your Wazuh indexer node.

Run the following command:

```bash
curl -k -u admin:admin https://<WAZUH_INDEXER_IP_ADDRESS>:9200
```

A successful response returns JSON containing information such as:

- Node name
- Cluster name and UUID
- OpenSearch version
- Lucene version

Example response structure:

```json
{
  "name": "node-1",
  "cluster_name": "wazuh-cluster",
  "cluster_uuid": "<cluster-uuid>",
  "version": {
    "number": "7.10.2"
  },
  "tagline": "The OpenSearch Project: https://opensearch.org/"
}
```

> **Important:** The example version and node details are illustrative. Verify the actual output from your installation.

### 3. Check Cluster Health

Run the following command to inspect the indexer nodes:

```bash
curl -k -u admin:admin "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cat/nodes?v"
```

To inspect cluster health:

```bash
curl -k -u admin:admin "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cluster/health?pretty"
```

Check the returned node list and cluster health information to confirm that the cluster is operating as expected.

### 4. Troubleshooting

If initialization or testing fails:

1. Confirm that the Wazuh Indexer service is running.
2. Verify the configured node IP address and certificate paths.
3. Check the service logs:

   ```bash
   sudo journalctl -u wazuh-indexer -n 50 --no-pager
   ```

4. Investigate authentication, certificate, or connectivity errors before changing configuration.

> **Security note:** `admin:admin` is shown only to match the documentation example. Use the actual configured credentials for your installation, and do not publish passwords, private keys, or certificate archives in a public GitHub repository.

## Reference

- [Official Wazuh 4.12 — Installing the Wazuh indexer step by step](https://documentation.wazuh.com/4.12/installation-guide/wazuh-indexer/step-by-step.html)


# 2.Wazuh Server Node Installation

**Wazuh Version:** 4.12.0  
**Components:** Wazuh Manager, Filebeat  
**Reference:** [Official Wazuh Documentation — Step-by-Step Installation](https://documentation.wazuh.com/4.12/installation-guide/wazuh-server/step-by-step.html)

## 1. Add the Wazuh Repository

Install the required packages:

```bash
sudo apt-get install gnupg apt-transport-https
```

Import the Wazuh GPG key:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```

Add the repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
```

Update package information:

```bash
sudo apt-get update
```

> If the Wazuh repository is already configured and enabled on the same host, do not add it again.

## 2. Install Wazuh Manager and Filebeat

Install Wazuh Manager:

```bash
sudo apt-get -y install wazuh-manager=4.12.0-1
```

Install Filebeat:

```bash
sudo apt-get -y install filebeat=7.10.2-1
```

## 3. Configure Filebeat

Download the Wazuh Filebeat configuration:

```bash
sudo curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.12/tpl/wazuh/filebeat/filebeat.yml
```

Edit the configuration:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Configure the Wazuh Indexer address:

```yaml
output.elasticsearch:
  hosts: ["192.168.0.107:9200"]  [Replace with ur ip]
  protocol: https
  username: ${username}
  password: ${password}
```

Replace `<WAZUH_INDEXER_IP>` with the actual Indexer IP address. Preserve the rest of the downloaded configuration.

## 4. Configure the Filebeat Keystore

Create the keystore:

```bash
sudo filebeat keystore create
```

Add the Indexer credentials:

```bash
echo admin | sudo filebeat keystore add username --stdin --force
echo admin | sudo filebeat keystore add password --stdin --force
```

> `admin:admin` is the default example credential in the step-by-step documentation. Use your actual configured credentials if they have been changed.

## 5. Install the Wazuh Alerts Template and Module

Download the alerts template:

```bash
sudo curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.12.0/extensions/elasticsearch/7.x/wazuh-template.json
sudo chmod go+r /etc/filebeat/wazuh-template.json
```

Install the Wazuh Filebeat module:

```bash
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz | sudo tar -xvz -C /usr/share/filebeat/module
```

## 6. Deploy Filebeat Certificates

Ensure that `wazuh-certificates.tar`, generated during the initial configuration, is available in your working directory.

Set the certificate name used in `config.yml`:

```bash
NODE_NAME=wazuh-1 [check  cat config.tml ]
```

Create the certificate directory:

```bash
sudo mkdir -p /etc/filebeat/certs
```

Extract the required certificates:

```bash
sudo tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
```

Rename the certificates:

```bash
sudo mv -n /etc/filebeat/certs/$NODE_NAME.pem /etc/filebeat/certs/filebeat.pem
sudo mv -n /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem
```

Set the recommended permissions and ownership:

```bash
sudo chmod 500 /etc/filebeat/certs
sudo chmod 400 /etc/filebeat/certs/*
sudo chown -R root:root /etc/filebeat/certs
```

**Important:** Replace `<SERVER_NODE_NAME>` with the exact server node certificate name. Check that the expected files exist before proceeding.

## 7. Configure the Wazuh Indexer Connection

This step is required if you want to use the Wazuh vulnerability detection capability.

Store the Indexer credentials in the Wazuh Manager keystore:

```bash
echo 'admin' | sudo /var/ossec/bin/wazuh-keystore -f indexer -k username
echo 'admin' | sudo /var/ossec/bin/wazuh-keystore -f indexer -k password
```
-admin:admin username and password [ u can change it]
Edit the Manager configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Configure the `<indexer>` section to use the actual Indexer address and the Filebeat certificate files:

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.0.107:9200</host> [Replace with ur indexer ip]
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/filebeat/certs/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
    <key>/etc/filebeat/certs/filebeat-key.pem</key>
  </ssl>
</indexer>
```

Replace `<WAZUH_INDEXER_IP>` with the actual Indexer IP or hostname. Ensure that the certificate paths match the files deployed in the previous step.

For multiple Indexer nodes, add a `<host>` entry for each node.

## 8. Start the Wazuh Manager

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
```

Verify its status:

```bash
sudo systemctl status wazuh-manager --no-pager
```

## 9. Start Filebeat

Enable and start Filebeat:

```bash
sudo systemctl daemon-reload
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

Test the Indexer connection:

```bash
sudo filebeat test output
```

A successful connection should report that the connection and TLS handshake succeeded and show the Filebeat output version.

## 10. Disable Wazuh Repository Updates

Only after completing the installation of **all required Wazuh components**, disable the repository to prevent accidental upgrades:

```bash
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

## 11. Troubleshooting

Check the Wazuh Manager:

```bash
sudo systemctl status wazuh-manager --no-pager
sudo journalctl -u wazuh-manager -n 50 --no-pager
```

Check Filebeat:

```bash
sudo systemctl status filebeat --no-pager
sudo journalctl -u filebeat -n 50 --no-pager
sudo filebeat test output
```

---

# Wazuh Server Cluster Configuration

> This section applies to **multi-node deployments**. A single-node lab does not need a master/worker cluster.

## 12. Configure the Master Node

Edit the Wazuh Manager configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Configure the `<cluster>` section:

```xml
<cluster>
  <name>wazuh</name>
  <node_name>master-node</node_name>
  <node_type>master</node_type>
  <key>REPLACE_WITH_32_CHARACTER_CLUSTER_KEY</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
  <nodes>
    <node><WAZUH_MASTER_ADDRESS></node>
  </nodes>
  <hidden>no</hidden>
  <disabled>no</disabled>
</cluster>
```

Generate a cluster key:

```bash
openssl rand -hex 16
```

Use the same generated key on every node. Replace `<WAZUH_MASTER_ADDRESS>` with the master node's actual IP address or DNS name.

Restart the Manager:

```bash
sudo systemctl restart wazuh-manager
```

## 13. Configure Worker Nodes

On each worker, edit:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Configure its `<cluster>` section:

```xml
<cluster>
  <name>wazuh</name>
  <node_name>worker-node</node_name>
  <node_type>worker</node_type>
  <key>REPLACE_WITH_32_CHARACTER_CLUSTER_KEY</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
  <nodes>
    <node><WAZUH_MASTER_ADDRESS></node>
  </nodes>
  <hidden>no</hidden>
  <disabled>no</disabled>
</cluster>
```

Use a unique `node_name` for every worker. Use the same cluster key as the master and specify the master's address.

Restart the Manager on each worker:

```bash
sudo systemctl restart wazuh-manager
```

## 14. Verify the Cluster

Run on a Wazuh server node:

```bash
sudo /var/ossec/bin/cluster_control -l
```

The output should list the connected master and worker nodes, their roles, versions, and addresses.

**Security notes:**
- Do not commit passwords, private keys, or `wazuh-certificates.tar` to a public GitHub repository.
- Use real IP addresses and node names only in private configuration files, not as assumed example values.
- Open the required network ports between components according to the official Wazuh documentation.


# 3.Wazuh Dashboard Installation

**Wazuh Version:** 4.12.0  
**Package:** `wazuh-dashboard=4.12.0-1`  
**Reference:** [Official Wazuh Dashboard Installation Documentation](https://documentation.wazuh.com/4.12/installation-guide/wazuh-dashboard/step-by-step.html)

## 1. Install Dependencies

```bash
sudo apt-get install debhelper tar curl libcap2-bin
```

Install the repository dependencies if they are missing:

```bash
sudo apt-get install gnupg apt-transport-https
```

## 2. Add the Wazuh Repository

Import the GPG key:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```

Add the repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
```

Update the package information:

```bash
sudo apt-get update
```

> If the Wazuh repository is already configured and enabled on this host, do not add it again.

## 3. Install the Wazuh Dashboard

```bash
sudo apt-get -y install wazuh-dashboard=4.12.0-1
```

Verify the installed package version:

```bash
dpkg-query -W -f='${Version}\n' wazuh-dashboard
```

## 4. Configure the Dashboard

Open the configuration file:

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Configure the relevant settings:

```yaml
server.host: 0.0.0.0 [ listen all the ip]
server.port: 443
opensearch.hosts: ["https://192.168.0.107:9200"] [ Replace with indexer ip]
opensearch.ssl.verificationMode: certificate
```

Replace `<WAZUH_INDEXER_IP>` with the actual IP address or hostname of the Wazuh Indexer.

**Important:** Preserve the other existing settings in the file. Do not replace the entire configuration with this snippet. Confirm that the configured Indexer address matches the deployed certificates and network setup.

## 5. Deploy Dashboard Certificates

Ensure that `wazuh-certificates.tar`, generated during the initial configuration, is available in your working directory.

Set the dashboard certificate name used in `config.yml`:

```bash
NODE_NAME=dashboard [ check config.yml ]
```

Create the certificate directory:

```bash
sudo mkdir -p /etc/wazuh-dashboard/certs
```

Extract the required certificates:

```bash
sudo tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
```

Rename the certificate files:

```bash
sudo mv -n /etc/wazuh-dashboard/certs/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo mv -n /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
```

Set permissions and ownership:

```bash
sudo chmod 500 /etc/wazuh-dashboard/certs
sudo chmod 400 /etc/wazuh-dashboard/certs/*
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

Check the files before proceeding:

```bash
sudo ls -l /etc/wazuh-dashboard/certs/
```

Expected files:

- `dashboard.pem`
- `dashboard-key.pem`
- `root-ca.pem`

> **Caution:** The `mv -n` command does not overwrite an existing destination file. If a rename does not happen, inspect the directory before taking further action. Do not delete or move existing certificates blindly.

## 6. Start the Dashboard Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
```

Check the service status:

```bash
sudo systemctl status wazuh-dashboard --no-pager
```

Check recent logs if necessary:

```bash
sudo journalctl -u wazuh-dashboard -n 50 --no-pager
```

## 7. Configure the Wazuh API Connection

Edit the Wazuh dashboard configuration:

```bash
sudo nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

Configure the Wazuh server master node address:

```yaml
hosts:
  - default:
      url: https://192.168.0.107 [ Replace with server ip]
      port: 55000
      username: wazuh-wui
      password: <WAZUH_WUI_PASSWORD>
      run_as: false
```

Replace the server address and password with the actual configured values.

> The default `wazuh-wui` credentials shown in the official installation guide are examples. If the password has been changed, update this configuration accordingly.

## 8. Access the Wazuh Dashboard

Open the dashboard in a browser:

```text
https://192.168.0.107 [ Replace with dashboard ip]
```

The default step-by-step installation credentials are:

- **Username:** `admin`
- **Password:** `admin`

Use these credentials only if they have not been changed.

The browser may warn that the certificate is not issued by a trusted authority. For a lab, you can verify the certificate and configure trust for the generated root CA. Do not ignore certificate errors without checking the certificate and endpoint.

## 9. Secure the Installation

After confirming the central components work, change the default credentials according to the official Wazuh password-management documentation.

For an all-in-one deployment, the documented password tool is:

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --api --change-all --admin-user wazuh --admin-password wazuh
```

**Important security notes:**

- Use appropriate administrative credentials for your installation.
- The command above uses example administrative credentials; do not treat `wazuh:wazuh` as a secure production password.
- Password changes can require updates to the dashboard, Filebeat, and other components.
- Store generated passwords securely. Never commit credentials to GitHub.

## 10. Disable Wazuh Repository Updates

Only after installing all required Wazuh components, disable the repository to avoid accidental upgrades:

```bash
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

## 11. Troubleshooting

Check the service:

```bash
sudo systemctl status wazuh-dashboard --no-pager
```

Inspect recent logs:

```bash
sudo journalctl -u wazuh-dashboard -n 100 --no-pager
```

Check the Indexer endpoint:

```bash
curl -k -u admin https://<WAZUH_INDEXER_IP>:9200
```

The command prompts for the password instead of placing it directly in the command line.

Common issues include:

- **HTTP 500:** Inspect dashboard logs for the underlying error.
- **Authentication errors:** Verify the Indexer credentials and dashboard configuration.
- **SSL or certificate errors:** Check certificate paths, permissions, trust configuration, and hostname/IP compatibility.
- **Connection refused or timeout:** Verify that the Indexer is running, the address is correct, and required network ports are reachable.

---


