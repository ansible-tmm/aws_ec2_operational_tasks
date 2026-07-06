# AWS EC2 Operational Tasks

This Ansible Project showcase multiple AWS (Amazon Web Services) operational tasks fully automated with Ansible Playbooks.

An operational tasks is a routine task an operator (cloud administer) has to do outside of provisioning and deprovisioning resources.  Declarative automation (such as AWS CloudFormation templates) are great until someone manually, or some tool outside of CloudFormation starts interacting with the public cloud.  There is always use cases for imperative repeatable tasks that operators are doing manually.

**Table of Contents**
* [Glossary of AWS terms](#glossary-of-aws-terms)
* [Ansible Playbook Examples](#ansible-playbook-examples)
   * [Retrieve and Stop](#retrieve-and-stop)
   * [Turn long-running instances off](#turn-long-running-instances-off)
   * [Turn untagged instances off](#turn-untagged-instances-off)
   * [Retrieve instances without a specific tag](#retrieve-instances-without-a-specific-tag)
   * [Put instances to sleep](#put-instances-to-sleep)
   * [Wake up sleepy instances](#wake-up-sleepy-instances)
* [Provision on Ansible Automation Platform](#provision-on-ansible-automation-platform)

# Glossary of AWS terms

**ec2** - Amazon Elastic Compute Cloud, Secure, resizable compute capacity in the cloud.

**ec2 region** - Amazon cloud computing resources are hosted in multiple locations world-wide. These locations are composed of AWS Regions, Availability Zones, and Local Zones. Each AWS Region is a separate geographic area. Each AWS Region has multiple, isolated locations known as Availability Zones. [read more here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html)

**ec2 instance** - Any compute deployment within the Amazon EC2 service.

**tag** -  metadata for AWS resources. Each tag is a simple label consisting of a customer-defined key and an optional value that can make it easier to manage, search for, and filter resources by purpose, owner, environment, or other criteria. AWS tags can be used for many purposes.

# Ansible Playbook Examples

## Retrieve and Stop

`playbooks/stop_instances.yaml` [link](playbooks/stop_instances.yaml)


```yaml
ansible-playbook stop_instances.yaml -e "your_region=us-west-1"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` and stop them.

## Turn long-running instances off

`playbooks/turn_off_time.yaml` [link](playbooks/turn_off_time.yaml)


```yaml
ansible-playbook turn_off_time.yaml -e "your_region=us-west-1 kill_time=100"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` that have been running over 100 minutes, then stop them.

## Turn untagged instances off

`playbooks/no_tags.yaml` [link](playbooks/no_tags.yaml)


```yaml
ansible-playbook no_tags.yaml -e "your_region=us-west-1"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` that have no tags, and stop them. No tags means literally they have zero tags, not a single tag.  It is not looking for a specific tag.

## Retrieve instances without a specific tag

`playbooks/missing_tag.yaml` [link](playbooks/missing_tag.yaml)


```yaml
ansible-playbook missing_tag.yaml -e "your_region=us-west-1"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` that don't have the specific tag key `owner` (e.g. tags.owner).  This allows an operator to enforce a specific tag (e.g. assign an owner to each resource in this example) or it will be [scheduled](https://docs.ansible.com/ansible-tower/latest/html/userguide/scheduling.html) in Ansible Automation Platform to be turned off.

See [Provision on Ansible Automation Platform](#provision-on-ansible-automation-platform) to create a workflow that runs this playbook across multiple AWS regions in parallel.

## Put instances to sleep

`playbooks/sleep_schedule_off.yaml` [link](playbooks/sleep_schedule_off.yaml)


```yaml
ansible-playbook sleep_schedule_off.yaml -e "your_region=us-west-1"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` that have the specific tag key pair `sleep_schedule: true`.  This allows operators to optionally add a tag to their instances, to turn them off at night.  This Ansible Playbook would be [scheduled](https://docs.ansible.com/ansible-tower/latest/html/userguide/scheduling.html) in Ansible Automation Platform to run every evening at a specific time.  In a multi-region scenario this could be enhanced with timezones to allow operators to specify their timezone or working hours.

## Wake up sleepy instances

`playbooks/sleep_schedule_on.yaml` [link](playbooks/sleep_schedule_on.yaml)


```yaml
ansible-playbook sleep_schedule_on.yaml -e "your_region=us-west-1"
```

This Ansible Playbook will retrieve all instances from the specified region `us-west-1` that have the specific tag key pair `sleep_schedule: true`.  This allows operators to optionally add a tag to their instances, to turn them on in the morning (opposite of previous example, please see above for more information).

# Provision on Ansible Automation Platform

Use `provision_demo.yml` to create the project, job template, and workflow on Ansible Automation Platform (AAP). This follows the same configuration-as-code pattern used in [ansible/product-demos](https://github.com/ansible/product-demos), with resource definitions in `setup.yml` and provisioning handled by `provision_demo.yml`.

The workflow runs `playbooks/missing_tag.yaml` in parallel across every AWS region listed in `setup.yml` (17 regions enabled on the default commercial partition). Each workflow node passes `your_region` as an extra variable and uses the region code as the node alias.

```
START ─┬─ us-east-1
       ├─ us-east-2
       ├─ us-west-1
       ├─ ... (one node per region in aws_regions)
       └─ ap-southeast-2
```

## Prerequisites

1. Install required collections:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

2. An AWS credential must already exist in AAP for the target organization. Update `aap_aws_credential_name` in `setup.yml` or override it at runtime.

3. A service account token (or OAuth token) with permission to create projects, job templates, and workflows in the target organization.

## Authentication

`provision_demo.yml` authenticates with a Bearer token. Username and password are not required.

Export the AAP hostname and token before running the playbook:

```bash
export AAP_SERVER="aap-nostromo.demoredhat.com"
export MY_SERVICE_TOKEN="your-service-token"
```

Supported environment variables:

| Variable | Purpose |
| --- | --- |
| `AAP_SERVER` | Platform gateway hostname (preferred) |
| `MY_SERVICE_TOKEN` | Service account or OAuth token (preferred) |
| `CONTROLLER_HOST` | Fallback hostname if `AAP_SERVER` is not set |
| `CONTROLLER_OAUTH_TOKEN` | Fallback token if `MY_SERVICE_TOKEN` is not set |

If both `AAP_SERVER` and `CONTROLLER_HOST` are set, `AAP_SERVER` takes precedence. Unset `CONTROLLER_HOST` when it points at a different AAP instance than the one you want to provision.

For environments such as `*.demoredhat.com` that use a self-signed certificate, the playbook sets `aap_validate_certs: false` by default.

## Provision the demo

```bash
ansible-playbook provision_demo.yml
```

Override defaults from `setup.yml` with extra vars when needed:

```bash
ansible-playbook provision_demo.yml \
  -e aap_organization=Default \
  -e aap_aws_credential_name='AWS -sean'
```

## Resources created

| AAP resource | Name |
| --- | --- |
| Project | AWS EC2 Operational Tasks |
| Inventory | AWS EC2 Operational Tasks Inventory |
| Job template | AWS | EC2 | Missing Owner Tag |
| Workflow job template | AWS | EC2 | Enforce Owner Tag \| All Regions |

After provisioning, launch **AWS \| EC2 \| Enforce Owner Tag \| All Regions** from the AAP UI. You can also schedule that workflow to run periodically so untagged instances are stopped automatically in every configured region.

## Configuration files

- `setup.yml` — AAP resource definitions, including AWS regions and workflow nodes
- `provision_demo.yml` — playbook that creates or updates those resources via the AAP API

To add or remove regions, edit the `aws_regions` list in `setup.yml`, then re-run `provision_demo.yml`. Workflow nodes are generated automatically from that list.

# Ansible Demos

![ansible demo logo](images/Ansible-Demo-Logo.png)

This project is maintained by Red Hat.  

- For more information on Ansible Automation please visit
[https://www.ansible.com/](https://www.ansible.com/)
- For more Ansible Automation Platform demos, please visit
[https://github.com/ansible/product-demos](https://github.com/ansible/product-demos)
- Please consider subscribing to us on YouTube:
[https://youtube.com/ansibleautomation](https://youtube.com/AnsibleAutomation)
