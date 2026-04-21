# ansible-roboshop-roles-tf

The `ansible-pull` configuration layer for the Terraform-provisioned RoboShop platform. Each EC2 instance runs `ansible-pull` on first boot against this repo and applies the role that matches its component.

![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![Jinja](https://img.shields.io/badge/Jinja2-B41717?logo=jinja&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?logo=amazonaws&logoColor=white)

## Overview

This is the Ansible companion to [`terraform-aws-roboshop`](https://github.com/sashank1064/terraform-aws-roboshop). Terraform provisions and tags the host. User data runs:

```bash
ansible-pull -U https://github.com/sashank1064/ansible-roboshop-roles-tf.git \
  -e component=$1 -e env=$2 main.yaml
```

`main.yaml` is a minimal dispatcher that applies the role whose name matches the `component` extra-var:

```yaml
- name: configuring "{{ component }}"
  become: yes
  hosts: localhost
  connection: local
  roles:
    - "{{ component }}"
```

That's it. One repo, one playbook, eleven roles, zero control-plane infrastructure.

## Roles

| Role | What it installs |
|---|---|
| `common` | Base packages, hardening, CloudWatch agent, shared users |
| `mongodb` | MongoDB service and config for the catalogue and user services |
| `redis` | Redis for user sessions and cart data |
| `mysql` | MySQL for the shipping service |
| `rabbitmq` | RabbitMQ broker for payment and dispatch |
| `catalogue` | Node.js catalogue service |
| `user` | Node.js user service |
| `cart` | Node.js cart service |
| `shipping` | Java shipping service |
| `payment` | Python payment service |
| `frontend` | Nginx reverse proxy for the web tier |

## Why ansible-pull, not a control node

- **No bastion-from-bastion.** No "first provision the Ansible server, then have it reach 20 other hosts over SSH."
- **Self-healing on recreate.** If Terraform destroys and recreates an instance (scale-in, drift repair, AMI refresh), the new host pulls and applies its role on its own.
- **State lives with the host.** Each instance owns its configuration run; there's no single node whose failure stalls the platform.
- **Dev-loop is fast.** Merge to this repo, terminate a single instance, the ASG or Terraform recreates it and it picks up the change.

## Repo layout

```
.
├── main.yaml           # dispatcher: apply role matching {{ component }}
├── inventory.ini       # only used for ad-hoc runs outside ansible-pull
└── roles/
    ├── common/
    ├── mongodb/
    ├── redis/
    ├── mysql/
    ├── rabbitmq/
    ├── catalogue/
    ├── user/
    ├── cart/
    ├── shipping/
    ├── payment/
    └── frontend/
```

Each role follows the standard skeleton (`tasks/`, `handlers/`, `templates/`, `defaults/`, `vars/`, `meta/`).

## Running outside Terraform

For debugging a role on a host that's already up:

```bash
# On the target host
sudo ansible-pull -U https://github.com/sashank1064/ansible-roboshop-roles-tf.git \
  -e component=catalogue -e env=dev main.yaml
```

Or run the role locally against `localhost`:

```bash
ansible-playbook -i inventory.ini main.yaml -e component=catalogue -e env=dev
```

## Related repos

1. [`terraform-aws-vpc`](https://github.com/sashank1064/terraform-aws-vpc), [`terraform-aws-securitygroup`](https://github.com/sashank1064/terraform-aws-securitygroup), [`terraform-aws-instance`](https://github.com/sashank1064/terraform-aws-instance): foundational modules
2. [`terraform-aws-roboshop`](https://github.com/sashank1064/terraform-aws-roboshop): the Terraform side, which calls ansible-pull in user data
3. [`roboshop-infra-dev`](https://github.com/sashank1064/roboshop-infra-dev): phased platform deployment
4. `ansible-roboshop-roles-tf` (this repo)
5. [`ansible-roboshop-roles`](https://github.com/sashank1064/ansible-roboshop-roles): the push-based variant using a control node
