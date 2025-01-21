# Homeserver

My personal home server with ansible.

# Prerequisites

- install ansible

  ```bash
  ~ sudo dnf install ansible
  ```

- obtain secrets.yaml and vault-key.sh from keepass

# Execute main playbook

```bash
~ ansible-playbook -i inventory.yml run.yaml --ask-become-pass --vault-pass-file vault-key.sh
```
