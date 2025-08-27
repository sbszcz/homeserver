# Homeserver

My personal home server with ansible.

# Prerequisites

- install ansible

  ```bash
  ~ sudo dnf install ansible
  ```

- obtain secrets.yaml and vault-key.sh from keepass
- install dependencies
  
  ```bash
  ~ ansible-galaxy install -r requirements.yml

  ```

# Execute main playbook

```bash
~ ansible-playbook -i inventory.yml run.yml --ask-become-pass --vault-pass-file vault-key.sh
```
