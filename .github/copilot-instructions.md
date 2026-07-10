# Copilot-Instruktionen für das Homeserver-Projekt

Dieses Repository automatisiert die Einrichtung und den Betrieb eines privaten
Homeservers mit **Ansible**. Container-Anwendungen werden als **Komodo-Stacks**
(Docker Compose) über [Komodo](https://komo.do) per GitOps ausgerollt. Ansible
installiert Komodo und bereitet die Host-Umgebung vor; die eigentlichen Stacks
liegen unter `komodo/` und werden von Komodo direkt aus diesem Repository
deployt. Bitte halte dich bei allen Änderungen an die hier beschriebenen
Konventionen.

## Überblick

- **Zweck:** Provisionierung eines Debian/Ubuntu-basierten Homeservers.
- **Ziel-Host:** `homie` (siehe `inventory.yml`), Zugriff via SSH-Key
  `~/.ssh/id_maintainer`.
- **Haupt-Playbook:** `run.yml` führt die Rollen in fester Reihenfolge aus.
- **Deployment der Stacks:** Komodo überwacht das Repository und rollt die
  Compose-Stacks unter `komodo/stacks/` per GitOps aus.
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
- `roles/` – eine Rolle pro Aufgabe bzw. Host-Vorbereitung eines Stacks.
- `komodo/komodo.toml` – Komodo-ResourceSync: deklariert das Repo und die Stacks
  (`[[stack]]`-Einträge).
- `komodo/stacks/<name>/compose.yaml` – Docker-Compose-Definition je Stack, von
  Komodo ausgerollt.
- `mise.toml` – Task-Runner-Definitionen (`ping`, `run-playbook`/`rp`, `lint`,
  `facts`).

## Rollen-Konventionen

Es gibt zwei Arten von Rollen:

1. **System-Rollen** (`system_prepare`, `ssh_prepare`, `docker`, `komodo`) –
   bereiten Host, Docker und Komodo vor. Die Rolle `komodo` installiert Komodo
   (Core + Periphery) via `community.docker.docker_compose_v2` und legt eine
   ResourceSync an, die auf `komodo/komodo.toml` zeigt
   (siehe `roles/komodo/tasks/create_resource_sync.yml`).
2. **Prepare-Rollen** (`*_prepare`, z. B. `immich_prepare`, `paperless_prepare`,
   `homeassistant_prepare`) – bereiten den Host für einen Stack vor (Mounts,
   Datenordner, Secret-Dateien). Das eigentliche Deployment des Containers
   übernimmt Komodo, nicht Ansible.

Beim Anlegen oder Ändern einer **Prepare-Rolle** gilt das etablierte Muster:

- `meta/main.yml` deklariert bei Bedarf Abhängigkeiten (z. B. `system_handlers`
  für Reload-Handler).
- `defaults/main.yml` setzt die rollen-spezifischen Variablen (Pfade, Mounts,
  Secret-Verzeichnisse).
- `tasks/main.yml` folgt diesem Ablauf:
  1. Optionale Vorbereitung: Mount-Punkte anlegen und NFS-Mounts sicherstellen.
  2. Daten- und Unterordner unter `/{{ container_data }}/<stack>` anlegen
     (Schleifen über Unterordner via `loop` + `loop_control: loop_var:
     sub_directory`).
  3. Secrets-Ordner (`owner`/`group` `root`, `mode: "0700"`) anlegen und die
     Secret-Dateien aus dem Vault schreiben (`no_log: true`).

Der Stack selbst wird **nicht** über eine Ansible-Rolle deployt, sondern als
Komodo-Stack unter `komodo/stacks/<name>/compose.yaml` gepflegt und in
`komodo/komodo.toml` als `[[stack]]`-Eintrag registriert. Komodo überwacht das
Repository (`poll_for_updates = true`) und rollt Änderungen per GitOps auf dem
Server `homie` aus.

## Coding-Konventionen

- Jede YAML-Datei beginnt mit `---`.
- Ansible-Templates erhalten **keine** `.j2`-Endung, sondern behalten die Endung
  der Zieldatei (z. B. `compose.yaml`, `compose.env`, `core.config.toml`).
  Ansible verarbeitet die Quelle ohnehin als Jinja2, daher ist die Endung reine
  Konvention (siehe `roles/komodo/templates/`).
- Compose-Dateien der Komodo-Stacks unter `komodo/stacks/<name>/compose.yaml`
  sind statische Docker-Compose-Dateien (kein Jinja2); dynamische Werte werden
  über Umgebungsvariablen bzw. Host-seitig geschriebene Secret-Dateien eingebunden.
- Tasks haben aussagekräftige `name`-Felder; bei Stack-Bezug wird der Stack-Name
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
  `environment`. Muster (analog zur Komodo-Installation, `roles/komodo`, und den
  Stacks unter `komodo/stacks/`, z. B. `paperless`):
  - Im Compose-File einen `secrets:`-Block mit `file:`-Quelle definieren (die
    Datei liegt unter `/{{ container_data }}/<stack>/secrets/<name>`) und den
    Diensten per `secrets:`-Liste zuweisen.
  - Im Container die zugehörige `<NAME>_FILE`-Variable auf `/run/secrets/<name>`
    setzen (z. B. `POSTGRES_PASSWORD_FILE`, `PAPERLESS_DBPASS_FILE`), sofern das
    Image diese `_FILE`-Konvention unterstützt.
  - Die Secret-Dateien schreibt Ansible in der Prepare-Rolle aus dem Vault auf den
    Host (Task-Muster `Write <stack> docker secrets`): `owner`/`group` `root`, das
    Secrets-Verzeichnis mit `mode: "0700"`, die Dateien mit `no_log: true`.
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
`ansible.builtin.uri`-Aufrufe (Komodo-API) im `--check`-Modus nicht funktionieren.

## Neue Anwendung hinzufügen (Checkliste)

1. Host-Vorbereitung als Rolle `roles/<name>_prepare/` mit `meta/`, `defaults/`,
   `tasks/` anlegen (Mounts, Datenordner, Secret-Dateien; bestehende Prepare-Rolle
   wie `immich_prepare` als Vorlage nehmen).
2. Compose-Definition unter `komodo/stacks/<name>/compose.yaml` erstellen.
3. Stack in `komodo/komodo.toml` als `[[stack]]` registrieren
   (`server = "homie"`, `linked_repo = "homeserver"`,
   `run_directory = "komodo/stacks/<name>"`, `file_paths = ["compose.yaml"]`).
4. Benötigte Secrets in `secrets.yml` (+ `.example`) und projektweite Werte in
   `common.yml` ergänzen.
5. Prepare-Rolle in `run.yml` eintragen.
6. Mit `mise run lint` prüfen.

## Planungsmodus

Wenn ausdrücklich um einen Plan gebeten wird (Stichworte: "Plan", "planen",
"nur analysieren"):

- Ändere keinen Code und lege keine Dateien an.
- Analysiere den relevanten Codebestand und benenne betroffene Rollen/Dateien.
- Stelle offene Fragen, wenn Anforderungen unklar sind.
