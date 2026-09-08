# LiteLLM Ansible Role

This role deploys LiteLLM with Docker Compose on a target host. It uses a bundled compose template, creates the required environment and config files, mounts the generated config into the container, and starts the service on a configurable host port.

## What the role does

- Validates the required role inputs
- Creates the deployment and config directories
- Generates or reuses LiteLLM master and salt keys
- Renders the bundled LiteLLM Docker Compose template (no external download required)
- Writes a `.env` file for the container runtime
- Renders a LiteLLM config YAML from `slm_ansible_role_litellm_config_yaml`
- Creates a compose override to mount the config file and expose the public port
- Starts the service with `community.docker.docker_compose_v2`
- Provides a container health check against `http://localhost:4000`

## Requirements

- Ansible 2.2+
- Docker on the target host
- Docker Compose v2 support
- The `community.docker` collection installed in the Ansible environment

## Required variables

The role expects these values to be set for each deployment:

- `slm_ansible_role_litellm_docker_compose_dir`: directory for the LiteLLM compose files
- `slm_ansible_role_litellm_config_dir`: directory for the generated LiteLLM config YAML

## Optional variables

- `slm_ansible_role_litellm_version`: LiteLLM image tag, default `v1.98.0`
- `slm_ansible_role_litellm_config_yaml`: dict used to generate the LiteLLM proxy config file, see also: `https://docs.litellm.ai/docs/proxy/configs`, default `{}`
- `slm_ansible_role_litellm_public_port`: host port mapped to the container, default `4000`
- `slm_ansible_role_litellm_compose_project_name`: Docker Compose project name; when set, passed as `project_name` to the `docker_compose_v2` module — useful when LiteLLM is part of a larger Compose stack, default `""` (Compose derives the name from the directory)
- `slm_ansible_role_litellm_service_networks`: dict rendered verbatim under `services.litellm.networks:` in the compose override — use to attach LiteLLM to named networks and assign aliases, default `{}`
- `slm_ansible_role_litellm_compose_networks`: dict rendered verbatim under the top-level `networks:` key in the compose override — use to declare external networks the service should join, default `{}`
- `slm_ansible_role_litellm_master_key`: pre-defined LiteLLM master key; takes priority over an existing `.env` on the target host and over auto-generation, default `""` (key is read from existing `.env` or generated)
- `slm_ansible_role_litellm_salt_key`: pre-defined LiteLLM salt key; same priority rules as `slm_ansible_role_litellm_master_key`, default `""`

## Role outputs

- `slm_ansible_role_litellm_resolved_master_key`: the effective master key used for this deployment — set as a host variable after the role runs, marked `no_log: true` so it does not appear in Ansible output; use it in subsequent tasks to configure other services that need to call LiteLLM
- `slm_ansible_role_litellm_resolved_salt_key`: the effective salt key used for this deployment — same behaviour as `slm_ansible_role_litellm_resolved_master_key`

## Example usage

```yaml
- hosts: servers
  become: true
  vars:
    slm_ansible_role_litellm_docker_compose_dir: /opt/litellm/deployment
    slm_ansible_role_litellm_config_dir: /etc/litellm/config
    slm_ansible_role_litellm_public_port: 4000
    slm_ansible_role_litellm_config_yaml:
      model_list:
        - model_name: gpt-4o-mini
          litellm_params:
            model: openai/gpt-4o-mini
            api_key: "{{ lookup('env', 'OPENAI_API_KEY') }}"
      general_settings:
        master_key: "sk-..."
    slm_ansible_role_litellm_compose_project_name: my-litellm-stack
    slm_ansible_role_litellm_service_networks:
      my-litellm-stack_default:
        aliases:
          - litellm
          - llm-gateway
    slm_ansible_role_litellm_compose_networks:
      my-litellm-stack_default:
  roles:
    - role: slm-ansible-role-litellm
```

This is the same structure used in the Molecule scenario for local validation.

## Files created on the target host

When the role runs, it creates these files and directories:

- `${slm_ansible_role_litellm_docker_compose_dir}/`
- `${slm_ansible_role_litellm_docker_compose_dir}/litellm.yml`
- `${slm_ansible_role_litellm_docker_compose_dir}/litellm.override.yml`
- `${slm_ansible_role_litellm_docker_compose_dir}/.env`
- `${slm_ansible_role_litellm_config_dir}/litellm_config.yaml`

The service is exposed on the configured host port, with the default internal container port set to `4000`.

## Health and verification

The role configures a Docker health check that probes `http://localhost:4000` from inside the container.

The Molecule verification also checks the public API endpoint with Ansible `uri` and requires the response to be a successful HTTP status, so a test passes only if the service answers without a failure code.

## License

MIT-0

## Author

Benjamin Goetz
