# Flatcar Provisioning Automation for OVHcloud

This repository provides tools to automate Flatcar provisioning on [OVHcloud][ovhcloud] using [Terraform][terraform].

## Features

- Minimal configuration required (demo deployment works with default settings w/o any customisation, just run `terraform apply`!).
- Deploy one or multiple servers.
- Per-server custom configuration via separate [container linux config][container-linux-config] files.

## Prerequisites

1. OVHcloud credentials: username, tenant name, password, authentication URL, region.

## Disclaimer

This Terraform code is tested against a [DevStack][devstack] instance, for actual OVHcloud deployment the "network" attribute might need to be modified.

## HowTo

This will create a server in 'RegionOne' using a medium instance size in the 'public' network with 'default' security group.
See "Customisation" below for advanced settings.

1. Clone the repo.
2. Add credentials in a `terraform.tfvars` file, expected credentials name can be found in `provider.tf`
3. Run
   ```shell
   terraform init
   ```
4. Edit [`server1.yaml`][server-1] and provide your own custom provisioning configuration in [container linux config][container-linux-config] syntax.
5. Plan and apply.
   This will also auto-generate an SSH key pair to log in to the server after provisioning.
   The public key of that key pair will be registered with OpenStack. <br />
   Invoke Terraform:
   ```shell
   terraform plan
   terraform apply
   ```

Terraform will print server information (name, ipv4 and v6) after deployment concluded.
The deployment will create an SSH key pair in `.ssh/`.

After provisioning concluded the private key of the key pair generated during provisioning can be used to connect to node. As mentioned in the disclaimer, the tested setup relies on DevStack so we need to proxy jump on the devstack instance before reaching the deployed instance:
```shell
ssh -J user@[DEVSTACK-IP] -i ./.ssh/provisioning_private_key.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null core@[SERVER-IP]
```

_NOTE_:
* Server IP address can be found at any moment after deployment by running `terraform output`
* If you update server configuration(s) in `server-configs` and re-run `terraform apply`, the instance will be **replaced**.
Consider adding [`create_before_destroy`](https://www.terraform.io/docs/configuration/meta-arguments/lifecycle.html#syntax-and-arguments) to the `openstack_compute_instance_v2` resource in [`compute.tf`](compute.tf) to avoid services becoming unavailable during reprovisioning.
* SSH key is passed twice: in the Butane configuration and in the `key_pair` argument of the instance - while the SSH keys in the Butane configuration are added by Ignition during the first boot of the instance, `key_pair` are processed by `coreos-metadata-sshkeys@.service` (a.k.a [`afterburn`][afterburn]) from the OpenStack [metadata service][metadata-service]. It means that you can install SSH keys without Ignition and change them as wanted.

### Customisation

The provisioning automation can be customised via settings in [`terraform.tfvars`][terraform.tfvars]:
  - `cluster_name`: Descriptive name of your cluster, will be used to generate server names.
    `flatcar-terraform` by default.
  - `machines`: Add more machines to your deployment.
    Each machine name must be unique and requires a respective `[NAME].yaml` server configuration in [`server-configs`](server-configs).
    An example / default configuration for machine `server1` is provided with [`server1.yaml`](server-configs/server1.yaml).
    During provisioning, server names are generated by concatenating the cluster name and the machine name - the defaults above will create a single server named `flatcar-terraform-server1`.
  - `ssh_keys`: Additional SSH public keys to add to core user's `authorized_keys`.
    Note that during provisioning, a provisioning RSA key pair will be generated and stored in the local directory `.ssh/provisioning_private_key.pem` and `.ssh/provisioning_key.pub`, respectively.
    The private key can be used for ssh connections (`core` user) to all provisioned servers.
  - `release_channel`: Select one of "lts", "stable", "beta", or "alpha".
    Read more about channels [here](https://www.flatcar.org/releases).
  - `flatcar_version`: Select the desired Flatcar version for the given channel (default to "current", which is the latest).
  - `flavor_name`: The spec of the machine, it default to ds1G (1vCPU, 10GB of disk and 1GB of memory)
  - `ssh`: A boolean to create and attach a security group to the instances to allow SSH connections. (_NOTE_: At this moment, when creating an instance with this variable enabled then turning it off does not work as the security group is not firstly detached from the instance so the security group can't be deleted)

[afterburn]: https://coreos.github.io/afterburn/
[container-linux-config]: https://www.flatcar.org/docs/latest/provisioning/config-transpiler/configuration/
[devstack]: https://opendev.org/openstack/devstack
[metadata-service]: https://docs.openstack.org/nova/zed/user/metadata.html
[ovhcloud]: https://www.ovhcloud.com/
[server-1]: server-configs/server1.yaml
[terraform]: https://www.terraform.io/
[terraform-tfvars]: terraform.tfvars