# Vault/Consul Cluster Deployment Automation with Terraform and Ansible.

- Infrastructure provisioned by Terraform Enterprise or OSS [this code]
- Configured by Brian Shumate Ansible Roles [see below]

For Consul, Vault Ansible roles check:

- https://github.com/brianshumate/ansible-consul/
- https://github.com/brianshumate/ansible-vault/

## Google Cloud setup

If you aren't familiar with Google Cloud you can read our [Getting Starting guide](GCP.md) to setup the required GCP project, service account and get the necessary credentials.

## Terraform Enteprise (TFE) Workspace

Once your GCP environment is ready, reach [Terraform Enterprise UI](https://app.terraform.io) to create a Workspace linked to a fork of this repository in your organisation.

### Terraform Variables

You need to setup the following required Terraform variables in your workspace, as in this example

      region: europe-west1
      region_zone: europe-west1-c
      project_name: sb-vault
      ssh_pub_key: <YOUR_SSH_PUBLIC_KEY>
      gcp_dns_zone: vault-prod
      gcp_dns_domain: prod.yet.org.

Make sure you update the variable above according to your needs. You can also look inside `variable.tf` to see what other ones you can update.

### Terraform OSS

By the way if you provision your environment with Terraform OSS instead, make sure you uncomment the `account_file_path` variable in both this file and `main.tf`. And then setup all the variables values including your GCP credentials file location in `terraform.tfvars`.

### Terraform Environment Variable

For security concerns it's better to setup the Google Credentials in an Environment variable. So create a sensitive environment variable like this

      GOOGLE_CREDENTIALS: <JSON_KEY_ON_A_SINGLE_LINE>

Make sure you join all the lines of your JSON or Terraform Enterprise will complain, he doesn't accept newlines in environment variables.

## Terraform Plan

You're now ready to run a plan, click on `Queue Plan` in TFE UI. Or run `terraform plan` if you use OSS.

## Terraform Apply

Once the plan ends succesfully, you can click on `Confirm & Apply` in TFE UI. Or run `terraform apply`.

## Outputs

If you're lucky you should have an output giving you ip addresses of consul and vault nodes.

    Outputs:
    
    Vault IPs = [
        35.X33.6.X3,
        35.X95.59.X7
    ]
    consul IPs = [
        35.X89.235.X90,
        35.X41.245.X6,
        35.X41.130.X4
    ]

We deliberately refrain ourselves to use modules to keep everything in a single repository to make it more readable and to ease troubleshooting.

## Ansible workflow

To create a cluster you first have to create an Inventory files with your nodes, for example:

    [consul_instances]
    c3.prod.yet.org consul_node_name=c1 consul_client_address="{{ consul_bind_address }}" consul_node_role=bootstrap  
    c2.prod.yet.org consul_node_name=c2 consul_client_address="{{ consul_bind_address }}" consul_node_role=server
    c1.prod.yet.org consul_node_name=c3 consul_client_address="{{ consul_bind_address }}" consul_node_role=server
    v1.prod.yet.org consul_node_name=cc1
    v2.prod.yet.org consul_node_name=cc2

    [vault_instances]
    v1.prod.yet.org
    v2.prod.yet.org

And tell Ansible what to do in `site.yml` like this

    - name: Configure Consul cluster
      hosts: consul_instances
      any_errors_fatal: true
      become: true
      become_user: root
      roles:
        - {role: brianshumate.consul}
      vars:
        ansible_ssh_user: sebastien
        consul_iface: ens4
        consul_install_remotely: true
    
    - name: Install Vault
      hosts: vault_instances
      any_errors_fatal: true
      become: true
      become_user: root
      roles:
        - {role: brianshumate.vault}
      vars:
        ansible_ssh_user: <USERNAME>
        vault_iface: ens4
        vault_install_remotely: true
        vault_ui: true
        vault_tls_disable: false
        vault_tls_src_files: <PATH_OF_YOUR_CERT_FILES>
        validate_certs_during_api_reachable_check: false

Once the infrastructrure is provisioned with Terraform, lastly to configure your cluster, just run:

    ansible-playbook -i hosts site.yml
