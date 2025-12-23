📘 Pipeline Configuration Guide

This document describes the structure and purpose of the pipeline configuration file. It explains all required and optional fields, how vault metadata works, how Ansible environments are resolved, and how stages are executed.

🧩 Top-Level Structure

Your configuration file contains the following major sections:

vault_metadata:        # Vault secret structure
ansible_environment:   # Global Ansible environment
stages:                # Pipeline stages


Below is a full explanation of each section.

🔐 1. vault_metadata (Required)

Defines how the pipeline fetches secrets from Vault.

Structure
vault_metadata:
  vault_secret_path: <vault secret path>
  vault_variable_path: <vault variable path>
  vault_keys:
    - name: <vault key name>
      type: <type of secret>

Field Breakdown
Key	Required	Description
vault_secret_path	✔️ Yes	Path to location where host credentials are stored in Vault (SSH user, host, private key).
vault_variable_path	✔️ Yes	Path where variable groups (per-stage variables) are stored.
vault_keys	✔️ Yes	A list describing which secrets are expected from Vault.
vault_keys[].name	✔️ Yes	Key name exactly as stored in Vault.
vault_keys[].type	✔️ Yes	Type of secret — used for grouping or validation (host, user, ssh, etc.).
Example (Your Provided)
vault_metadata:
  vault_secret_path: secret/data/core/machine
  vault_variable_path: secret/data/core/variable
  vault_keys:
    - name: ec_host
      type: host
    - name: ec_user
      type: user
    - name: ec_ssh
      type: ssh


Purpose:
The pipeline reads these values from Vault and injects them into Ansible as environment variables or connection parameters.

🛠️ 2. ansible_environment (Required)

Defines the global/default Ansible environment used by all stages unless overridden at the stage level.

Structure
ansible_environment:
  inventory: <path>
  ansible_cfg: <path>
  host_vars: <path>
  group_vars: <path>
  collections:
    - namespace.collection_name

Field Breakdown
Key	Required	Description
inventory	✔️ Yes	Path to main inventory file.
ansible_cfg	✔️ Yes	Path to ansible.cfg.
host_vars	❌ Optional	Directory containing host-specific variable files.
group_vars	❌ Optional	Directory containing group-specific variable files.
collections	❌ Optional	Ansible collections to be installed before running playbooks. Must be FQCN format.
Example (Yours)
ansible_environment:
  inventory: pipeline_config/inventories/inventory.yml
  ansible_cfg: pipeline_config/ansible.cfg
  host_vars: pipeline_config/inventories/host_vars
  group_vars: pipeline_config/inventories/group_vars
  collections:
    - ibm.ibm_zos_core

🚀 3. stages (Required)

Defines the stage workflow to execute.
Stages can run in parallel unless they explicitly specify dependencies (e.g., runAfter).

🧱 Stage Structure
stages:
  - name: <stage name>
    description: <description>
    vault_variable_key: <optional key to fetch variables from the given Vault variable path>
    ansible_environment: <optional environment override>
    playbooks: [list of playbook files]
    cleanup: [list of playbooks to run after stage]
    validation: [list of playbooks to validate stage outcome]

Stage Keys Explained
Key	Required	Description
name	✔️ Yes	Unique identifier of the stage.
description	✔️ Yes	Human-readable purpose of the stage.
playbooks	✔️ Yes	Playbooks to run during this stage.
vault_variable_key	❌ Optional	Name of variable group stored in Vault—for stage-specific parameters.
ansible_environment	❌ Optional	Overrides global environment for this stage.
cleanup	❌ Optional	Playbooks executed after main execution to cleanup.
validation	❌ Optional	Playbooks to confirm stage success.
✅ Stage Example — Provided in Your Config
Stage 1: "Ping the EC"
- name: Ping the EC
  description: Test the ssh connection to EC using ping
  playbooks:
    - zos_concepts/zos_ping/zos_ping.yaml
  cleanup:
    - zos_concepts/zos_ping/zos_ping.yaml
  validation:
     - zos_concepts/zos_ping/zos_ping.yaml

Explanation

No vault_variable_key → Uses only global vault secrets.

Uses global Ansible environment.

Validation and cleanup use the same playbook.

Stage 2: "Archive and Unarchive"

This stage overrides the Ansible environment and uses its own variable group.

- name: Archive and Unarchive
  description: "Transfer Data Sets Using Ansible"
  vault_variable_key: archive_vars
  ansible_environment:
    inventory: zos_concepts/data_transfer/archive_copy_unarchive_restore/inventories/inventory.yml
    ansible_cfg: zos_concepts/zos_ping/ansible.cfg
    host_vars: pipeline_config/inventories/host_vars
    group_vars: pipeline_config/inventories/group_vars
    collections:
      - ibm.ibm_zos_core
  playbooks:
    - zos_concepts/data_transfer/archive_copy_unarchive_restore/archive_fetch_data_sets.yml
    - zos_concepts/data_transfer/archive_copy_unarchive_restore/unarchive_data_sets.yml

Explanation

vault_variable_key: archive_vars

The pipeline retrieves additional variables (e.g., dataset names, HLQs, etc.) from Vault under this key.

Stage-level Ansible environment

Inventory is overridden for this stage.

Collections also overridden (still same, but can be different).

Two playbooks executed in sequence.

📝 Notes & Best Practices
✔️ Use stage-level ansible_environment sparingly

Only override when a stage truly requires a different inventory or config.

✔️ Always use FQCN for collections

Example: ibm.ibm_zos_core

✔️ Vault keys should only contain names, not values

✔️ Keep playbooks short and meaningful

Each stage should perform a clear atomic unit of work.

📘 Final Combined Example

Here is the final representation combining everything:

vault_metadata:
  vault_secret_path: secret/data/core/machine
  vault_variable_path: secret/data/core/variable
  vault_keys:
    - name: ec_host
      type: host
    - name: ec_user
      type: user
    - name: ec_ssh
      type: ssh

ansible_environment:
  inventory: pipeline_config/inventories/inventory.yml
  ansible_cfg: pipeline_config/ansible.cfg
  host_vars: pipeline_config/inventories/host_vars
  group_vars: pipeline_config/inventories/group_vars
  collections:
    - ibm.ibm_zos_core

stages:
  - name: Ping the EC
    description: Test the ssh connection to EC using ping
    playbooks:
      - zos_concepts/zos_ping/zos_ping.yaml
    cleanup:
      - zos_concepts/zos_ping/zos_ping.yaml
    validation:
      - zos_concepts/zos_ping/zos_ping.yaml
  
  - name: Archive and Unarchive
    description: "Transfer Data Sets Using Ansible"
    vault_variable_key: archive_vars
    ansible_environment:
      inventory: zos_concepts/data_transfer/archive_copy_unarchive_restore/inventories/inventory.yml
      ansible_cfg: zos_concepts/zos_ping/ansible.cfg
      collections:
        - ibm.ibm_zos_core
    playbooks:
      - zos_concepts/data_transfer/archive_copy_unarchive_restore/archive_fetch_data_sets.yml
      - zos_concepts/data_transfer/archive_copy_unarchive_restore/unarchive_data_sets.yml