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

