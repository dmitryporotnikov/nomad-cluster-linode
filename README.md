# Nomad cluster on Linode

[![Terraform](https://img.shields.io/badge/Terraform-infra-7B42BC?logo=terraform&logoColor=white)](infra)
[![Nomad](https://img.shields.io/badge/Nomad-1.x-00CA8E?logo=hashicorp&logoColor=white)](https://developer.hashicorp.com/nomad)
[![Linode](https://img.shields.io/badge/Linode-Akamai-00A95C)](https://www.linode.com/)

Terraform and Ansible for a six-node Nomad cluster on Linode: three servers, three clients, Consul on all of them, and a private VPC between the nodes.

This follows Justin Mitchel's tutorial and repo, with the Nomad, Consul and Linode providers bumped to current versions. The original walkthrough is still the best explanation of why the pieces are arranged this way:

- Video: https://www.youtube.com/watch?v=D8KgwFFNFWA
- Repo: https://github.com/jmitchel3/nomad-cluster-terraform-ansible-linode

## What you get

Three Nomad servers handle scheduling and cluster state. Three clients run the workloads, with Docker and the CNI plugins installed. Consul provides service discovery and health checks on every node, and everything talks over a VPC on `10.0.0.0/24`.

```
┌─────────────────────────────────────────────────────────┐
│                    Linode VPC (10.0.0.0/24)             │
│                                                         │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
│   │ Nomad Srv 1 │   │ Nomad Srv 2 │   │ Nomad Srv 3 │   │
│   │ Consul Srv  │   │ Consul Srv  │   │ Consul Srv  │   │
│   │ 10.0.0.5    │   │ 10.0.0.6    │   │ 10.0.0.7    │   │
│   └─────────────┘   └─────────────┘   └─────────────┘   │
│                                                         │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
│   │ Nomad Cli 1 │   │ Nomad Cli 2 │   │ Nomad Cli 3 │   │
│   │ Consul Cli  │   │ Consul Cli  │   │ Consul Cli  │   │
│   │ 10.0.0.13   │   │ 10.0.0.14   │   │ 10.0.0.15   │   │
│   └─────────────┘   └─────────────┘   └─────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Before you start

A Linode account and an API token. Put the token in `infra/terraform.tfvars`:

```bash
echo 'linode_api_token="your_api_token"' >> infra/terraform.tfvars
```

You'll also need Terraform and Python 3 on the machine you run this from.

## Deploy

```bash
cd infra
terraform plan
terraform apply
```

That creates the instances, wires the VPC, and writes an Ansible inventory to `inventory.ini`.

```bash
cd ../config
python3 -m venv venv
source venv/bin/activate
pip install ansible
ansible-playbook -i inventory.ini main.yaml
```

Ansible installs Consul in server mode on the Nomad servers and client mode elsewhere, installs Nomad everywhere, puts Docker and CNI plugins on the clients, and starts the services.

## Check it and run something

```bash
nomad server members -address=http://10.0.0.5:4646
nomad run ../samples-jobs/hello-world.hcl
nomad job status hello-world
```

The Nomad UI is at `http://10.0.0.5:4646`, Consul at `http://10.0.0.5:8500`.

Sample jobs in `samples-jobs/`: `hello-world.hcl` (a Docker web server), `cfe-nginx.hcl` (nginx with HTTP health checks), `iac-python.hcl` (a Python app), `load-balancing.hcl` (nginx behind Consul service discovery).

## Files worth knowing

Terraform lives in `infra/`: `main.tf` for the provider, `cluster_instance.tf` for servers and clients, `cluster_vpc.tf` for the network, `cluster_inventory.tf` for the generated inventory, `variables.tf` for the knobs.

Ansible lives in `config/`: `main.yaml` is the playbook, with `consul/`, `nomad/`, `docker_install/` and `cni_plugins/` roles.

## Tear down

```bash
cd infra
terraform destroy
```

That removes the instances. Check the Linode dashboard afterwards to make sure nothing was left behind.

There's no license file in this repo yet, so treat it as all rights reserved.
