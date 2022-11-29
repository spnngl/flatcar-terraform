# Flatcar Provisioning Automation for Hetzner Cloud

This repository provides tools to automate Flatcar provisioning on [Hetzner Cloud](https://www.hetzner.com/cloud) using [Terraform](https://www.terraform.io/).

## Features

- Minimal configuration required (demo deployment works with default settings w/o any customisation, just run `terraform apply`!).
- Deploy one or multiple servers.
- Per-server custom configuration via separate [container linux config](https://www.flatcar.org/docs/latest/provisioning/config-transpiler/configuration/) files.


## Prerequisites

1. A Hetzner Cloud [API token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/).


## HowTo

This will create a server in Falkenstein using the smallest instance size available.
See "Customisation" below for advanced settings.

1. Clone the repo.
2. Run
   ```shell
   terraform init
   ```
3. Edit [`server1.yaml`](server-configs/server1.yaml) and provide your own custom provisioning configuration in [container linux config](https://www.flatcar.org/docs/latest/provisioning/config-transpiler/configuration/) syntax.
4. Plan and apply.
   This will also auto-generate an SSH key pair to log in to the server after provisioning.
   The public key of that key pair will be registered with Hetzner. <br />
   Optionally provide your Hetzner API token (see prerequisites above) in an environment variable.
   If it is not provided Terraform will prompt for the token on each run.
   ```shell
      export HCLOUD_TOKEN="..."
   ```
   Invoke Terraform:
   ```shell
   terraform plan
   terraform apply
   ```

Terraform will print server information (name, ipv4 and v6, and ID) after deployment concluded.
The deployment will create an SSH key pair in `.ssh/`.

After provisioning concluded the private key of the key pair generated during provisioning can be used to connect to nodes:
```shell
ssh -i ./.ssh/provisioning_private_key.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null core@[SERVER-IP]
```

**NOTE** that if you update server configuration(s) in `server-configs` and re-run `terraform apply`, the instance will be **replaced**.
Consider adding [`create_before_destroy`](https://www.terraform.io/docs/configuration/meta-arguments/lifecycle.html#syntax-and-arguments) to the `hcloud_server` resource in [`hetzner-machines.tf`](hetzner-machines.tf) to avoid services becoming unavailable during reprovisioning.

### Customisation

The provisioning automation can be customised via settings in [`terraform.tfvars`](terraform.tfvars):
  - `cluster_name`: Descriptive name of your cluster, will be used to generate server names.
    `flatcar` by default.
  - `machines`: Add more machines to your deployment.
    Each machine name must be unique and requires a respective `[NAME].yaml` server configuration in [`server-configs`](server-configs).
    An example / default configuration for machine `server1` is provided with [`server1.yaml`](server-configs/server1.yaml).
    During provisioning, server names are generated by concatenating the cluster name and the machine name - the defaults above will create a single server named `flatcar-server1`.
  - `location`: Hetzner Cloud region to deploy to. See here for a list of all locations: https://docs.hetzner.com/cloud/general/locations/
  - `server_type`: See https://www.hetzner.com/cloud for a list (scroll down).
    Add additional yaml configurations if you added `machines` in `terraform.tfvars`.
  - `ssh_keys`: Additional SSH public keys to add to core user's `authorized_keys`.
    Note that during provisioning, a provisioning RSA key pair will be generated and stored in the local directory `.ssh/provisioning_private_key.pem` and `.ssh/provisioning_key.pub`, respectively.
    The private key can be used for ssh connections (`core` user) to all provisioned servers.
  - `release_channel`: Select one of "lts", "stable", "beta", or "alpha".
    Read more about channels [here](https://www.flatcar.org/releases).