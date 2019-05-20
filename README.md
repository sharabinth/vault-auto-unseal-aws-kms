# Vault Auto Unseal Feature using AWS KMS Setup
This repo has a vagrant file to create an enterprise Vault Cluster with Consul backend.  
Seal stanza is added to the Vault config file to setup auto unseal using AWS KMS.
Vagrant file also creates a secondary cluster with auto unseal with AWS KMS using a different KMS ring.  This secondary can be configured as a DR or Performance Replication to perform further tests.

Each cluster contains 2 nodes and each note consists of a Consul Server and Vault server.  
The configuration is used for learning purposes.  This is NOT following the reference architecture for Vault and should not be used for a Production setup.

```
Cluster 1: Vault Primary cluster in DC1 

Cluster 2: Vault Secondary cluster in DC2 

```

All servers are set without TLS.

## Pre-Requisites
Create a folder named as ```ent``` and copy both the Consul and Vault enterprise binary zip files

```e.g., consul-enterprise_1.4.5+prem_linux_amd64.zip```

Rename ```data.txt.example``` to ```data.txt``` and update the values to specify the AWS related information such as Region, Access Key, Secret Key and AWS KMD Id.

## Vault Primary Cluster
2 node cluster is created with each node containing Vault and Consul servers. The server details are shown below

```
vault1   10.100.1.11
vault2   10.100.1.12
```

One of the Consul servers would become the leader.  Similarly one of Vault servers would become the Active node and the other node acts as Read Replica.

## Vault Secondary Cluster
2 node cluster is created with each node containing Vault and Consul servers. The server details are shown below

```
vault-dr1   10.100.2.11
vault-dr2   10.100.2.12
```

The vagrant file also has commented section to create another secondary cluster if required.  Check the content of the Vagrant file.

## Usage
If the ubuntu box is not available then it will take sometime to download the base box for the first time.  After the initial download, servers can be destroyed and recreated quickly with Vagrant

```
$vagrant up

$vagrant status

```

To check the status of the servers ssh into one of the nodes and check the cluster members and identify the leader.

```
$vagrant ssh vault1

vagrant@v1: $consul members

Node  Address           Status  Type    Build      Protocol  DC   Segment
v1    10.100.1.11:8301  alive   server  1.5.0+ent  2         dc1  <all>
v2    10.100.1.12:8301  alive   server  1.5.0+ent  2         dc1  <all>

vagrant@v1: $consul operator raft list-peers 

Node  ID                                    Address           State     Voter  RaftProtocol
v1    8c50f7de-634e-d7ee-17b8-7f904a34434d  10.100.1.11:8300  leader    true   3
v2    b3100f83-a4d1-89fd-5ab3-d96951e6a342  10.100.1.12:8300  follower  true   3

vagrant@v1: $consul info

vagrant@v1: $vault status

Key                      Value
---                      -----
Recovery Seal Type       awskms
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  n/a
HA Enabled               true

```

If ```vault status``` throws an error then check the AWS related information specified in the ```data.txt``` file.

Vault status would be shown as uninitialised and sealed.  By default the Recovery Seal Type is set to ```awskms```.

## Initialising and Unsealing Vault

Perform the following to initialise the Vault cluster and this should unseal both vault nodes due to ```awskms``` stanza.  
Initialisation is only required at one of the servers.

Vault can be initialised with Recovery keys.  In this case, the Recovery Seal Type would be set to ```Shamir```
If no recovery keys are requested then Recovery Seal Type would remain as ```awskms```
Having Recovery Keys would be useful as a last resort in case ```aws kms``` is accidently removed.

```
$vagrant ssh vault1

vagrant@v1: $vault status

vagrant@v1: $vault operator init -recovery-shares=1 recovery-threshold=1
Recovery Key 1: uf2dgR88QOHSSnnIDbSmGSNp1VJ+nIrdFlI0ZcebW80=

Initial Root Token: s.tlaekTPlcIytjcUWDGSEuqOh

Success! Vault is initialized

Recovery key initialized with 1 key shares and a key threshold of 1. Please
securely distribute the key shares printed above.

vagrant@v1: $vault status

Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    1
Threshold                1
Version                  1.1.2+prem
Cluster Name             vault-cluster-4ec938e4
Cluster ID               f395a1d9-fa91-4782-7aad-b7ea2a0b6af0
HA Enabled               true
HA Cluster               https://10.100.1.11:8201
HA Mode                  active
Last WAL                 16

vagrant@v1: $exit

$vagrant ssh vault2

vagrant@v2: $ vault status

```

## Testing Auto Unseal Feature

Once the Vault is initialised, it would be unsealed by the use of AWS KMS.   This is verified in the previous step.

When Vault is restarted, it would automatically unseal using AWS KMS.

```
vagrant@v1: $ sudo systemctl restart vault
vagrant@v1: $ sudo systemctl status vault

vault.service - "Vault secret management tool"
   Loaded: loaded (/etc/systemd/system/vault.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2019-05-20 07:45:16 UTC; 6s ago
 Main PID: 2679 (vault)
    Tasks: 11 (limit: 1152)
   CGroup: /system.slice/vault.service
           └─2679 /usr/local/bin/vault server -config=/etc/vault/vault.hcl

May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.817Z [INFO]  core: successfully mounted backend: type=identity path=identity/
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.817Z [INFO]  core: successfully mounted backend: type=cubbyhole path=cubbyhole/
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.825Z [INFO]  core: successfully enabled credential backend: type=token path=token/
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.825Z [INFO]  core: restoring leases
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.827Z [INFO]  expiration: lease restore complete
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.828Z [INFO]  identity: entities restored
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.828Z [INFO]  identity: groups restored
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.829Z [INFO]  mfa: configurations restored
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.830Z [INFO]  core: post-unseal setup complete
May 20 07:45:19 v1 vault[2679]: 2019-05-20T07:45:19.830Z [INFO]  replication.standby: requesting WAL stream: guard=dc8b0700



vagrant@v1: $ vault status

Key                                    Value
---                                    -----
Recovery Seal Type                     shamir
Initialized                            true
Sealed                                 false
Total Recovery Shares                  1
Threshold                              1
Version                                1.1.2+prem
Cluster Name                           vault-cluster-4ec938e4
Cluster ID                             f395a1d9-fa91-4782-7aad-b7ea2a0b6af0
HA Enabled                             true
HA Cluster                             https://10.100.1.12:8201
HA Mode                                standby
Active Node Address                    http://10.100.1.12:8200
Performance Standby Node               true
Performance Standby Last Remote WAL    0


vagrant@v1: $exit

$vagrant ssh vault2

vagrant@v2:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    1
Threshold                1
Version                  1.1.2+prem
Cluster Name             vault-cluster-4ec938e4
Cluster ID               f395a1d9-fa91-4782-7aad-b7ea2a0b6af0
HA Enabled               true
HA Cluster               https://10.100.1.12:8201
HA Mode                  active
Last WAL                 17

```

In the above, it should be noted that Node 1 has now become the Standby.  When Node 1 was restarted, Node 2 became the leader.

## Accessing UI

Use one of the server nodes to access the Consul UI on port 8500 and the Vault UI on port 8200.  The UI for Consul will not work if the leader is not elected.

e.g., Consul UI http://10.100.1.11:8500 

e.g., Vault UI http://10.100.2.11:8500 


