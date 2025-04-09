# Internet Solutions Ansible
Ansible playbooks for IS VM and application configuration.

## Getting Started
1. Follow your [os-specific instructions](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html) to install ansible on your workstation.
2. Install roles and collections from the `requirements.yml` file: `ansible-galaxy install -r requirements.yml`
3. Ensure you have credentials (AD, SSH key, password) any hosts you wish to connect to.
4. Ensure you have firewall rules in place that allow you to connect to remote hosts, or allow
your control node to connect to hosts.
5. Retrieve the [ansible vault password](https://console.cloud.google.com/security/secret-manager/secret/ansible-vault-secret/versions?project=prod-digital-is)
6. Place the vault password in the file `~/.vault_pass.txt`

### Updating Collections
See https://docs.ansible.com/ansible/latest/collections for collection versions.

### Testing
1. `pip3 install -r requirements.txt`
2. `molecule init role <role_name> -d gce`
3. Configure <role_name>/molecule/default/molecule.yml with required parameters.
4. `molecule test`

## GCP Firewall Rules for Ansible Controller
Ensure that the necessary ports (22,5985,5986) are open on the subnet(s) in which the controller and hosts live.

## Authentication
Playbooks execute on the controller will default to running as the `AD\ansible` domain user and expects to belong to the `Cloud Service Computer Administrators` and `Cloud Service Computer Remote Desktop Users` groups. Otherwise WinRM connectivity will fail with "invalid credentials" error.

## macOS multithreading security rules exception
Added security to restrict multithreading in macOS High Sierra and later versions of macOS cause the following error when executing ansible using the WinRM connector:
```
+[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
```
Set an environment variable .bash_profile (or .zshrc for recent macOS) to allow multithreading applications or scripts under the new macOS High Sierra security rules.
```
OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

## Dynamic Inventory Requirements
Leverages the [GCP dynamic inventory plugin](https://docs.ansible.com/ansible/latest/collections/google/cloud/gcp_compute_inventory.html) to extract from GCP a list of hosts and groups to which
roles and playbooks can be applied. Relies on [google-auth](https://pypi.org/project/google-auth/).

To list VMs in GCP you can execute: `ansible-inventory --list`

The `ansible.cfg` has been set to use `./inventories/gcp.yaml` as the default inventory. You may override this by passing a different inventory using `-i <inventory>` to `ansible-playbook`.

## Templating New Roles
Ansible roles should be templated using `ansible-galaxy`. For example: `ansible-galaxy init roles/is_web_server_base` will create the following:
```
roles/is_web_server_base
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

## Example Facts
```
"architecture": "64-bit",
"architecture2": "x86_64",
"distribution": "Microsoft Windows Server 2022 Datacenter",
"distribution_major_version": "10",
"distribution_version": "10.0.20348.0",
"netbios_name": "OSCONFIG-WINDOW",
"nodename": "osconfig-windows-host",
"os_family": "Windows",
"os_installation_type": "Server",
"os_name": "Microsoft Windows Server 2022 Datacenter",
"os_product_type": "server",
"windows_domain": "WORKGROUP",
"windows_domain_member": false,
"windows_domain_role": "Stand-alone server"
```

## Artifact Access
IS previously leveraged corporate artifactory to pull assets, however this is no longer accessible to us from GCP.
While iterating, all assets have been copied from artifactory to `gs://is_ops_artifacts`. Artifacts are downloaded
by Ansible onto the target VM using `gsutil`. The lab's Compute Engine default service account has been granted
`Storage Object Viewer`.

## Secrets
VMs require `roles/secretmanager.secretAccessor` and to use Secret Manager with workloads running on Compute Engine or GKE,
the underlying instance or node must have the [cloud-platform OAuth scope](https://cloud.google.com/secret-manager/docs/accessing-the-api#oauth-scopes).

```
--scopes "https://www.googleapis.com/auth/cloud-platform"
```

## Manually Joining a VM to AD
1. RDP to the VM and open a powershell session as an administrator
2. Execute: `iex((New-Object System.Net.WebClient).DownloadString('https://register-computer-6evp6cahmq-uc.a.run.app'))` to download and run the script from the cloud run autojoin service.
3. Reboot
