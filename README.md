# Rescue Vault from a broken raft setup

This is a simple demonstration of how to rescue Vault from a broken quorum from a bad setup (2 nodes) and finally what happens if you are following the suggested 3 or more node setup and one node fails.

This repo contains the code required for setting up a cluster using HashiCorp Vagrant. 
This also contains ansible playbooks to setup Vault and to forcibly destroy the quorum.

This is meant to be a purely educational experience, do not use this for production, or on a production cluster.

## Requirements

The setup expects VirtualBox hostnetwork to accept the following IP range: 192.168.56.10-192.168.56.12

- A virtual machine software (VirtualBox)
- HashiCorp Vagrant to
- Ansible
- Python 3
- Public ssh key saved in ~/.ssh/id_rsa.pem
- Strong enough machine to run up to 3 headless Ubuntu 22.04 servers

## Usage

Note: The Ansible code found under the 'ansible' directory can be used to configure Vault on any setup of your choosing, however, it is not meant for production use.

### Setup initial Cluster (ansible)

Start by running `vagrant up first second` inside the servers folder.
This will provision two servers and provision them using Ansible within the virtual machines. 

Once the cluster is up and running, jump into the ansible folder and run `ansible-playbook initialize_and_unseal_vault.yml`. 
This will do the following:

1. Start Vault on the first server
2. Initialize it with 1 key-share
3. Record the unseal-key and root key and save it to your local machine
4. Unseal the first Vault
5. Enable the KV secrets engine in the cluster
6. Start Vault on the second server
7. Unseal the second Vault
8. List the accessor of the KV secrets engine from both vm's to show that they are identical (to prove that Raft is up and running)

### Destroy your Raft Quorum (ansible)

With the cluster up and running from the previous stage, you will now commence by destroying your quorum in the cluster.
Since your current setup is using a 2 node setup, this will cause Vault to be inoperable and because we destroy the Raft database on the second node it will be as if the node has been completely corrupted. 

To start, from the ansible folder, run: `ansible-playbook cause_quorum_break.yml`
This will do the following on the second server:

1. Stop Vault
2. List the amount of folders found in the `/opt/vault/` folder (should be 2 from the start).
3. Delete the data folder and show that the `/opt/vault/` folder now only contains one folder (the tls certs generated on Vault installation).
4. It will then recreate the data folder and start Vault and try to unseal Vault. For user readability, the error is ignored but printed to the terminal.

This next step will happen on the first server:
5. It tries to list the Raft peers in the Vault cluster and prints the error

### Rescue your first server (manual)

The next steps are all manual, meant to be done to practice how to restore quorum.
This will all be done on the first server. SSH into it using `ssh vagrant@192.168.56.10`.

1. Create a json file inside the folder `/opt/vault/data/raft/`, name it `peers.json` with the following content (if you run it in another server setup):
```json
[
    {
    "id": "first",
    "address": "192.168.56.10:8201",
    "non_voter": false
    }
]
```
2. Restart Vault by using `sudo systemctl restart vault`
3. Save the root token to environment variables on the server by using the following commands: `export VAULT_TOKEN=$(cat root_token)`
4. Unseal the server using: `vault operator unseal $(cat unseal_keys | tr -d '[]"')`
5. Check that the server is up and running, this can be done either with `vault status` and noting that HA Mode is "active", or by using `vault operator raft list-peers`

### Rescue the second server (manual)
After this, you can rescue the second server by the following on the second server:
1. Stop Vault by using `sudo systemctl stop vault`. 
2. Delete the folder `/opt/vault/data/` and recreate it using `sudo install -g vault -o vault -d /opt/vault/data/`.
3. Start up Vault by using `sudo systemctl start vault` and wait a couple of seconds.
4. Unseal Vault by using `vault operator unseal` and the unseal key you can find in the file unseal_key.txt on your computer.
5. Log in to Vault using `vault login` and use the root_token you find in the root_token.txt file on your computer.
6. You can see that you have restored the second server to the cluster by using `vault operator raft list-peers` or `vault secrets list` and noting that the secret engine `kv` is enabled (from the inital setup).

## Vault setup following recommendation

Back in the servers folder, run `vagrant destroy --force` followed by `vagrant up`. This will start three servers with Vault installed. 
Once they are up and running, jump into the ansible folder and run `ansible-playbook initialize_and_unseal_vault.yml`.
Once the playbook is done, run: `ansible-playbook cause_quorum_break.yml`.

It will finally print the list of Vault Peers from the first server, this means that even though the second server lost all Raft data Vault was able to retain quorum and bring in the second server. 
In a real scenario when you don't just delete the `/opt/vault/data/` folder but the server is fully broken, you would remove the dead server using `vault operator raft remove-peer node-id` command. 