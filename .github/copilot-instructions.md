# Copilot-Instruktionen für das Homeserver-Projekt

Dieses Repository automatisiert die Einrichtung und den Betrieb eines privaten
Homeservers mit **Ansible**. Container-Anwendungen werden als **Portainer-Stacks**
(Docker Compose) über die Portainer-API ausgerollt. Bitte halte dich bei allen
Änderungen an die hier beschriebenen Konventionen.

## Überblick

- **Zweck:** Provisionierung eines Debian/Ubuntu-basierten Homeservers.
- **Ziel-Host:** `homie` (siehe `inventory.yml`), Zugriff via SSH-Key
  `~/.ssh/id_maintainer`.
- **Haupt-Playbook:** `run.yml` führt die Rollen in fester Reihenfolge aus.
- **Toolchain:** Über `mise.toml` werden Ansible-Version und Tasks verwaltet.

## Projektstruktur

- `run.yml` – Haupt-Playbook, bindet alle Rollen ein.
- `test.yml` – Hilfs-Playbook, zum Ausprobieren von Ansible.
- `inventory.yml` – Inventar mit dem Host `homie`.
- `requirements.yml` – externe Rollen (`geerlingguy.docker`) und Collections
  (`community.docker`, `ansible.posix`).
- `group_vars/all/common.yml` – projektweite, nicht geheime Variablen.
- `group_vars/all/secrets.yml` – **verschlüsselte** Geheimnisse (Ansible Vault),
  nicht eingecheckt. `secrets.yml.example` dient als Vorlage.
- `roles/` – eine Rolle pro Aufgabe bzw. Anwendungs-Stack.
- `mise.toml` – Task-Runner-Definitionen (`ping`, `run-playbook`/`rp`, `lint`,
  `facts`).

## Rollen-Konventionen

Es gibt zwei Arten von Rollen:

1. **System-Rollen** (`system_prepare`, `ssh_prepare`, `docker`, `portainer`) –
   bereiten Host, Docker und Portainer vor.
2. **Stack-Rollen** (`*_stack`, z. B. `immich_stack`, `pihole_stack`,
   `mosquitto_stack`, `paperless_stack`, `homeassistant_stack`) – rollen jeweils
   einen Portainer-Stack aus.

Beim Anlegen oder Ändern einer **Stack-Rolle** gilt das etablierte Muster:

- `meta/main.yml` deklariert die Abhängigkeit zur Rolle `portainer_stacks`.
- `defaults/main.yml` setzt mindestens `stack_name`.
- `templates/<name>.yml` enthält die Docker-Compose-Definition als Jinja2-Template.
- `tasks/main.yml` folgt diesem Ablauf:
  1. Optionale Vorbereitung (Mounts, Ordner, Konfig-Anpassungen).
  2. Authentifizierung/Prüfung via
     `include_role: portainer_stacks, tasks_from: authenticate_and_check_stack`.
  3. Anlegen der Stack-Datenordner unter
     `/home/{{ host_user }}/{{ container_data }}/{{ stack_name }}`
     (bedingt auf `jwt_token is defined and stack_not_exists`).
  4. Deployment via
     `include_role: portainer_stacks, tasks_from: create_stack` mit
     `stack_compose: "{{ lookup('ansible.builtin.template', '<name>.yml') }}"`
     und optional `stack_compose_env`.

Die Rolle `portainer_stacks` ist die geteilte Schnittstelle. Sie stellt die
Tasks `authenticate_and_check_stack` und `create_stack` bereit und nutzt die
generischen Variablen `stack_name`, `stack_compose` und `stack_compose_env`.
Diese generischen Namen sind bewusst ohne Rollen-Präfix (siehe `.ansible-lint`).

## Coding-Konventionen

- Jede YAML-Datei beginnt mit `---`.
- Templates erhalten **keine** `.j2`-Endung, sondern behalten die Endung der
  Zieldatei (z. B. `<name>.yml`, `<name>.env`). Ansible verarbeitet die Quelle
  ohnehin als Jinja2, daher ist die Endung reine Konvention (siehe bestehende
  Stack-Rollen wie `immich.yml`, `mosquitto.yml`).
- Tasks haben aussagekräftige `name`-Felder; bei Stack-Bezug wird `{{ stack_name }}`
  eingebunden.
- Immer **voll qualifizierte Modulnamen** verwenden
  (`ansible.builtin.*`, `ansible.posix.*`, `community.docker.*`).
- Für Schleifen über Ordner wird `loop` mit
  `loop_control: loop_var: sub_directory` genutzt.
- Datei-/Ordner-Tasks setzen `owner`, `group` (meist `{{ host_user }}`) und `mode`
  (typisch `"0755"`) als Strings.
- Zustandsändernde Systemdienste werden über `notify` + Handler
  (`handlers/main.yml`) neu gestartet, nicht direkt im Task.
- Idempotenz beachten: für Prüf-/Read-only-Abfragen `changed_when: false` setzen.
- Container-Daten liegen unter `{{ container_data }}` (Default: `container_data`),
  das via Bind-Mount nach `/{{ container_data }}` gespiegelt wird.

## Variablen und Geheimnisse

- **Keine** Geheimnisse im Klartext committen. Neue Secrets gehören in die
  vault-verschlüsselte `group_vars/all/secrets.yml`; ein Platzhalter wird zusätzlich
  in `secrets.yml.example` gepflegt.
- Beim Hinzufügen neuer Secrets trägt der Agent **ausschließlich Platzhalter** in
  `secrets.yml.example` ein. Die tatsächlichen, geheimen Werte legt **der Nutzer
  selbst** in der vault-verschlüsselten `group_vars/all/secrets.yml` an. Der Agent
  bearbeitet `secrets.yml` niemals eigenständig und **weist den Nutzer bei jeder
  Secret-Änderung ausdrücklich darauf hin**, die Werte dort selbst zu pflegen.
- Nicht-geheime, projektweite Werte gehören nach `group_vars/all/common.yml`.
- Der Vault-Schlüssel wird zur Laufzeit über `vault-key.sh` bereitgestellt
  (nicht eingecheckt).
- **Secrets in Containern nach Möglichkeit immer über die Docker-Compose-`secrets`-
  Variante mit `_FILE`-Umgebungsvariablen einbinden** – nicht als Klartext-Wert in
  `environment`. Muster (analog zur Komodo-Installation, `roles/komodo`):
  - Im Compose-Template einen `secrets:`-Block mit `file:`-Quelle definieren und den
    Diensten per `secrets:`-Liste zuweisen.
  - Im Container die zugehörige `<NAME>_FILE`-Variable auf `/run/secrets/<name>`
    setzen (z. B. `POSTGRES_PASSWORD_FILE`, `PAPERLESS_DBPASS_FILE`), sofern das
    Image diese `_FILE`-Konvention unterstützt.
  - Die Secret-Dateien schreibt Ansible aus dem Vault auf den Host (Task-Muster
    `Write <stack> docker secrets`): `owner`/`group` `root`, das Secrets-Verzeichnis
    mit `mode: "0700"`, die Dateien mit `no_log: true`.
  - Nur wenn ein Image die `_FILE`-Konvention nicht unterstützt, ausnahmsweise auf
    eine Klartext-`environment`-Variable ausweichen (Secret weiterhin aus dem Vault).


## Ausführung und Validierung

Bevorzugt die `mise`-Tasks:

- `mise run lint` – Linting mit `ansible-lint` (respektiert `.ansible-lint`).
- `mise run ping` – Erreichbarkeit des Hosts prüfen.
- `mise run rp` / `mise run run-playbook` – Haupt-Playbook ausführen.
- `mise run facts` – Ansible-Facts ausgeben.

Der Agent führt das Haupt-Playbook **niemals selbst** aus. `mise run rp` /
`mise run run-playbook` bzw. `ansible-playbook ... run.yml` werden
**ausschließlich vom Nutzer** ausgeführt. Der Agent darf `mise run lint` zur
Validierung nutzen, stößt aber **kein Deployment auf den Host** an.

Direkter Aufruf des Playbooks:

```bash
ansible-playbook -i inventory.yml run.yml --ask-become-pass --vault-pass-file vault-key.sh
```

Nach Änderungen an Rollen oder Playbooks möglichst `ansible-lint` laufen lassen.
Ein sauberer Check-Mode-Lauf ist derzeit nicht möglich, da einige
`ansible.builtin.uri`-Aufrufe (Portainer-API) im `--check`-Modus nicht funktionieren.

## Neue Anwendung hinzufügen (Checkliste)

1. Neue Rolle `roles/<name>_stack/` mit `meta/`, `defaults/`, `tasks/`,
   `templates/` anlegen (bestehende Stack-Rolle als Vorlage nehmen).
2. Compose-Template unter `templates/<name>.yml` erstellen.
3. Benötigte Secrets in `secrets.yml` (+ `.example`) und projektweite Werte in
   `common.yml` ergänzen.
4. Rolle in `run.yml` unter den bestehenden `*_stack`-Rollen eintragen.
5. Mit `mise run lint` prüfen.

## Planungsmodus

Wenn ausdrücklich um einen Plan gebeten wird (Stichworte: "Plan", "planen",
"nur analysieren"):

- Ändere keinen Code und lege keine Dateien an.
- Analysiere den relevanten Codebestand und benenne betroffene Rollen/Dateien.
- Stelle offene Fragen, wenn Anforderungen unklar sind.
