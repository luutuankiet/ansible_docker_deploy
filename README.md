# 🐳 Docker-in-Docker Provisioning Playbook

Provision isolated Docker-in-Docker (DinD) environments using Ansible. Each environment behaves like a pseudo-VM (or "VPC") with its own Docker Compose stack.

---

## 🚀 Features

* **Isolated Environments**: Each "VPC" is a containerized Docker daemon.
* **Layered Configuration**: Combine base infra + per-VPC extension templates.
* **Variable-Driven**: All configuration in a centralized YAML file.
* **Automated Deployment**: Ansible creates, configures, and launches each DinD container.

---

## 🔧 Requirements

* **Python 3** & `pip`
* **Ansible**:

  ```bash
  pip3 install ansible
  ```
* **Docker**: Installed on the control node (Ansible runs here).
* **Ansible Collections**:

  ```bash
  ansible-galaxy collection install community.docker
  ```

---

## ⚙️ Setup

1. **Install prerequisites** (see above)

2. **Clone the repo**:

   ```bash
   git clone <your_repo_url>
   cd your_repo_directory
   ```

3. **Configure your VPCs**:

   * Edit `quickstart/config/vps-config.yml`:

     ```yaml
     vpcs:
       ken-test:
         traefik_port: 60010
         dashboard_port: 60110
         memory: "1g"
         extension: "web"
       kha-test:
         traefik_port: 60011
         dashboard_port: 60111
         memory: "1g"
         extension: "web"
     ```
   * Optional: Create new extension templates in `quickstart/templates/extensions/`.

4. **Deploy**:

   ```bash
   ansible-playbook quickstart/playbook/deploy_docker.yaml
   ```

   Optional:

   * Limit to one VPC:

     ```bash
     ansible-playbook ... -e target_vpc=ken-test
     ```
   * Remove a VPC:

     ```bash
     ansible-playbook ... -e target_state=absent
     ```

---

## 🏗️ Architecture & Template Design

### 1. `vps-config.yml` — VPC definitions

Single source of truth for environments:

```yaml
vpcs:
  ken-test:
    traefik_port: 60010
    dashboard_port: 60110
    memory: "1g"
    extension: "web"
```

---

### 2. `base-infra.yml.j2` — Common services (Traefik, Wetty)

```yaml
traefik:
  image: traefik:v3.0
  # ...
wetty:
  image: wettyoss/wetty
  # ...
```

---

### 3. `extensions/*.yml.j2` — App-specific services

Example: `extensions/web.yml.j2`:

```yaml
webapp:
  image: tiangolo/uvicorn-gunicorn-fastapi:python3.9
  # ...
```

---

### 4. `final-compose.yml.j2` — Template merger

The playbook generates a final `docker-compose.yml` per VPC:

```yaml
- name: Generate final compose file (base + extension)
  template:
    src: "../templates/final-compose.yml.j2"
    dest: "./stacks/{{ vpc_item.key }}/docker-compose.yml"
  vars:
    vpc_name: "{{ vpc_item.key }}"
    extension_name: "{{ vpc_item.value.extension }}"
  loop: "{{ filtered_vpcs | dict2items }}"
```

---

### 5. Provisioning DinD containers

Each VPC is a DinD container:

```yaml
- name: Manage DinD container state (start or remove)
  community.docker.docker_container:
    name: "vpc-{{ vpc_item.key }}"
    image: "{{ common.image }}"
    state: "{{ state }}"
    ports:
      - "{{ vpc_item.value.traefik_port }}:80"
      - "{{ vpc_item.value.dashboard_port }}:8080"
    volumes:
      - "./stacks/{{ vpc_item.key }}:/app/stacks/{{ vpc_item.key }}:ro"
    command:
      - "docker"
      - "compose"
      - "-f"
      - "/app/stacks/{{ vpc_item.key }}/docker-compose.yml"
      - "up"
  loop: "{{ filtered_vpcs | dict2items }}"
```

---

## 📡 Accessing Your Environments

* Access via mapped ports:

  * `http://localhost:<dashboard_port>` → Traefik Dashboard
  * Other services routed internally by Traefik
* Manage lifecycle with:

  * `target_vpc` to limit
  * `target_state=absent` to stop/remove

---

## ♻️ Updating VPCs

1. Update image tags in `base-infra.yml.j2` or `extensions/*.yml.j2`
2. Modify `vps-config.yml` if needed
3. Redeploy:

   ```bash
   ansible-playbook quickstart/playbook/deploy_docker.yaml
   ```

---

## 🧹 Uninstall

Stop and remove all VPCs:

```bash
ansible-playbook quickstart/playbook/deploy_docker.yaml -e target_state=absent
```

Then optionally:

```bash
rm -rf ./stacks/
```