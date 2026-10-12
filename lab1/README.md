# Отчёт ЛР 1

## Цель работы

Научиться работать с Ansible на примере установки и настройки Opensearch на удалённом сервере.

# Структура проекта

Структура создаётся командами:

```bash
ansible-galaxy role init roles/opensearch
touch ansible.cfg playbook.yml README.md
```

Файл `ansible.cfg` задаёт инвентарь проекта, путь к ролям, проверку ключей SSH и отключает создание retry-файлов.

```
[defaults]
inventory = ./inventory.ini
roles_path = ./roles
host_key_checking = True
retry_files_enabled = False
```

`inventory.ini` задаёт список серверов, с которыми идёт взаимодействие.

```ini
[opensearch]
144.124.251.140 ansible_user=root
```

В процессе выполения работы встал вопрос, где взять сервер. Пришлось оплатить хостинг, для начала я оплатил русский сервер и планировал выпонлять работу на нём. Но потом выяснилось, что на сервере проблемы с выходом в интернет: почему то не проходило TLS-рукопожатие даже с google.com. Я сидел с нейронкой и пытался что-то предпринять, но в итоге пришёл к тому чтоб просто сменить сервер.

На этот раз я решил взять нидерландский сервер, чтобы проблем точно не возникало, пришлось оплатить иностранной картой :/ Но благо на этот раз всё работало, единственное что по SSH пришлось подключаться с VPN, ибо сервер уже был заблочен ТСПУ.

`playbook.yml` запускает роль opensearch на серверах из группы opensearch в inventory.ini:

```yaml
- name: Install and configure OpenSearch
  hosts: opensearch
  become: true

  roles:
    - opensearch
```

# Роль

### defaults/

Этот каталог предназначен для хранения переменных роли, которым задаются значения по умолчанию. Эти параметры можно переопределять при запуске Playbook или в других конфигурационных файлах Ansible. Переменные выносятся сюда для удобства, чтобы не приходилось каждый раз искать их для изменения значений.

#### `main.yml`:

```yaml
---
opensearch_version: "3.8.0"
opensearch_dashboards_version: "3.8.0"

opensearch_cluster_name: "ansible-cluster"
opensearch_node_name: "opensearch-node-1"
opensearch_network_host: "127.0.0.1"
opensearch_http_port: 9200
opensearch_discovery_type: single-node

opensearch_heap_size: "1g"
opensearch_max_map_count: 262144

opensearch_dashboards_server_host: "127.0.0.1"
opensearch_dashboards_server_port: 5601
opensearch_dashboards_opensearch_url: "https://127.0.0.1:9200"

opensearch_api_retries: 30
opensearch_api_delay: 10

opensearch_admin_username: admin
opensearch_admin_password: "{{ vault_opensearch_admin_password }}"

opensearch_dashboards_username: kibanaserver
opensearch_dashboards_password: "{{ vault_opensearch_dashboards_password }}"
```

### vars/

А этот каталог тоже содержит переменные, но предназначен для таких переменных, которые относятся к реализации роли и обычно не должны переопределяться так же свободно, как значения из `defaults/`.

#### `main.yml`:

```yaml
opensearch_service_name: opensearch
opensearch_dashboards_service_name: opensearch-dashboards

opensearch_config_path: /etc/opensearch
opensearch_dashboards_config_path: /etc/opensearch-dashboards

opensearch_repository_key_url: "https://artifacts.opensearch.org/publickeys/opensearch-release.pgp"
opensearch_repository_url: "https://artifacts.opensearch.org/releases/bundle/opensearch/3.x/apt"
opensearch_dashboards_repository_url: "https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.x/apt"
```

### handlers/

Каталог `handlers/` предназначен для обработчиков -- специальных задач Ansible, которые запускаются в ответ на уведомления от других задач. Например, смена конфигурации, которая требует рестарта сервиса.

#### `main.yml`

```yaml
- name: Restart OpenSearch
  ansible.builtin.systemd:
    name: "{{ opensearch_service_name }}"
    state: restarted

- name: Restart OpenSearch Dashboards
  ansible.builtin.systemd:
    name: "{{ opensearch_dashboards_service_name }}"
    state: restarted
```

### templates/

Этот каталог содержит шаблоны Jinja2, из которых Ansible создаёт конфигурационные файлы на удалённом сервере.

#### `jvm.options.j2` -- настройки джавы

```
-Xms{{ opensearch_heap_size }}
-Xmx{{ opensearch_heap_size }}
```

#### `opensearch.yml.j2` -- конфиг opensearch

```yaml
cluster.name: {{ opensearch_cluster_name }}
node.name: {{ opensearch_node_name }}

discovery.type: {{ opensearch_discovery_type }}

network.host: {{ opensearch_network_host }}
http.port: {{ opensearch_http_port }}

plugins.security.disabled: true
```

#### `opensearch_dashboards.yml.j2` -- конфиг opensearch dashboards

```yaml
server.host: {{ opensearch_dashboards_server_host }}
server.port: {{ opensearch_dashboards_server_port }}

opensearch.hosts:
  - "http://{{ opensearch_network_host }}:{{ opensearch_http_port }}"

```

### tasks/

Эта директория -- основное описание того, что Ansible должен сделать. В файле `main.yml` задачи расписываются в хронологическом порядке.

#### `main.py`:

1. Установка пакетов, необходимых для дальнейшей установки Opensearch:
```yaml
- name: Install required packages
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - apt-transport-https
    state: present
    update_cache: true
```

2. Создание каталога `/etc/apt/keyrings`, в котором будут храниться ключи для проверки пакетов:
```yaml
- name: Create APT keyrings directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"
```

3. Загрузка открытого ключа для подписи пакета Opensearch:
```yaml
- name: Download OpenSearch signing key
  ansible.builtin.get_url:
    url: "{{ opensearch_repository_key_url }}"
    dest: /tmp/opensearch-release.pgp
    mode: "0644"
```

4. Преобразование ключа в формат для apt:
```yaml
- name: Convert OpenSearch signing key to keyring format
  ansible.builtin.command:
    cmd: >-
      gpg --dearmor --batch --yes
      --output /etc/apt/keyrings/opensearch-release-keyring
      /tmp/opensearch-release.pgp
    creates: /etc/apt/keyrings/opensearch-release-keyring
```

5. Добавление репозитория Opensearch в список источников apt:
```yaml
- name: Add OpenSearch APT repository
  ansible.builtin.apt_repository:
    repo: >-
      deb [signed-by=/etc/apt/keyrings/opensearch-release-keyring]
      {{ opensearch_repository_url }} stable main
    filename: opensearch-3.x
    state: present
    update_cache: false
```

6. Подключение репозитория Opensearch Dashboards:
```yaml
- name: Add OpenSearch Dashboards APT repository
  ansible.builtin.apt_repository:
    repo: >-
      deb [signed-by=/etc/apt/keyrings/opensearch-release-keyring]
      {{ opensearch_dashboards_repository_url }} stable main
    filename: opensearch-dashboards-3.x
    state: present
    update_cache: false
```

7. Обновление списка доступных пакетов из подключённых репозиториев:
```yaml
- name: Update APT package cache
  ansible.builtin.apt:
    update_cache: true
```

8. Настройка системного параметра `vm.max_map_count`, необходимого для корректной работы OpenSearch:
```yaml
- name: Configure vm.max_map_count
  ansible.posix.sysctl:
    name: vm.max_map_count
    value: "{{ opensearch_max_map_count }}"
    state: present
    reload: true
```

9. Установка OpenSearch заданной версии с отключением демонстрационной конфигурации и плагина безопасности:
```yaml
- name: Install OpenSearch
  ansible.builtin.apt:
    name: "opensearch={{ opensearch_version }}"
    state: present
  environment:
    DISABLE_INSTALL_DEMO_CONFIG: "true"
    DISABLE_SECURITY_PLUGIN: "true"
```

10. Установка OpenSearch Dashboards заданной версии:
```yaml
- name: Install OpenSearch Dashboards
  ansible.builtin.apt:
    name: "opensearch-dashboards={{ opensearch_dashboards_version }}"
    state: present
```

11. Создание конфигурационного файла OpenSearch на основе шаблона Jinja2:
```yaml
- name: Configure OpenSearch
  ansible.builtin.template:
    src: opensearch.yml.j2
    dest: "{{ opensearch_config_path }}/opensearch.yml"
    owner: root
    group: opensearch
    mode: "0660"
  notify: Restart OpenSearch
```

12. Создание каталога `jvm.options.d` для хранения дополнительных параметров виртуальной машины Java:
```yaml
- name: Create JVM options directory
  ansible.builtin.file:
    path: "{{ opensearch_config_path }}/jvm.options.d"
    state: directory
    owner: root
    group: opensearch
    mode: "0755"
```

13. Создание файла с настройками JVM на основе шаблона `jvm.options.j2`:
```yaml
- name: Configure OpenSearch JVM heap
  ansible.builtin.template:
    src: jvm.options.j2
    dest: "{{ opensearch_config_path }}/jvm.options.d/heap.options"
    owner: root
    group: opensearch
    mode: "0644"
  notify: Restart OpenSearch
```

14. Создание конфигурационного файла OpenSearch Dashboards на основе шаблона Jinja2:
```yaml
- name: Configure OpenSearch Dashboards
  ansible.builtin.template:
    src: opensearch_dashboards.yml.j2
    dest: "{{ opensearch_dashboards_config_path }}/opensearch_dashboards.yml"
    owner: root
    group: opensearch-dashboards
    mode: "0660"
  notify: Restart OpenSearch Dashboards
```

15. Включение OpenSearch в автозапуск и запуск службы:
```yaml
- name: Enable and start OpenSearch
  ansible.builtin.systemd:
    name: "{{ opensearch_service_name }}"
    enabled: true
    state: started
```

16. Немедленное выполнение обработчиков, вызванных изменениями конфигурационных файлов:
```yaml
- name: Apply pending OpenSearch configuration changes
  ansible.builtin.meta: flush_handlers
```

17. Проверка доступности API OpenSearch. Ansible повторяет запросы, пока сервер не вернёт HTTP-код `200` или не закончатся попытки:
```yaml
- name: Wait for OpenSearch API
  ansible.builtin.uri:
    url: "http://127.0.0.1:{{ opensearch_http_port }}/"
    method: GET
    return_content: true
    status_code: 200
  register: opensearch_api
  retries: "{{ opensearch_api_retries }}"
  delay: "{{ opensearch_api_delay }}"
  until: opensearch_api.status == 200
```

18. Включение OpenSearch Dashboards в автозапуск и запуск службы:
```yaml
- name: Enable and start OpenSearch Dashboards
  ansible.builtin.systemd:
    name: "{{ opensearch_dashboards_service_name }}"
    enabled: true
    state: started
```

После написания роли её можно запустить при помощи плейбука:
```bash
ansible-playbook playbook.yml
```

В терминале мы видим применения каждой task-и по очереди и её статус:

```bash
seraypuh@seraypuh:~/Рабочий стол/labs-iac/lab1$ ansible-playbook playbook.yml

PLAY [Install and configure OpenSearch] ********************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install required packages] **************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Create APT keyrings directory] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Download OpenSearch signing key] ********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Convert OpenSearch signing key to keyring format] ***************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Add OpenSearch APT repository] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Add OpenSearch Dashboards APT repository] ***********************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Update APT package cache] ***************************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Configure vm.max_map_count] *************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install OpenSearch] *********************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install OpenSearch Dashboards] **********************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Configure OpenSearch] *******************************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Create JVM options directory] ***********************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Configure OpenSearch JVM heap] **********************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Configure OpenSearch Dashboards] ********************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Enable and start OpenSearch] ************************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Apply pending OpenSearch configuration changes] *****************************************************************************************************************************

RUNNING HANDLER [opensearch : Restart OpenSearch] **********************************************************************************************************************************************
changed: [144.124.251.140]

RUNNING HANDLER [opensearch : Restart OpenSearch Dashboards] ***********************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Wait for OpenSearch API] ****************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Enable and start OpenSearch Dashboards] *************************************************************************************************************************************
changed: [144.124.251.140]

PLAY RECAP *************************************************************************************************************************************************************************************
144.124.251.140            : ok=20   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

seraypuh@seraypuh:~/Рабочий стол/labs-iac/lab1$
```

Если что, некоторые задачи были уже в статусе `ok`, потому что в ходе выполнения я запускал в том числе и неудачные плейбуки, которые выполняли первые задачи, а затем заканчивали ошибкой.

Теперь проверяем что всё поднялось: запускаем ssh-туннель и заходим на страницу дашбордов:
```bash
ssh -L 5601:127.0.0.1:5601 root@<144.124.251.140>
```
![а](media/last.png)

Всё запустилось и работает!

Проверяем идемпотентность Ansible -- при повторном запуске ничего не меняется

```bash
seraypuh@seraypuh:~/Рабочий стол/labs-iac/lab1$ ansible-playbook playbook.yml

PLAY [Install and configure OpenSearch] ********************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install required packages] **************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Create APT keyrings directory] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Download OpenSearch signing key] ********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Convert OpenSearch signing key to keyring format] ***************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Add OpenSearch APT repository] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Add OpenSearch Dashboards APT repository] ***********************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Update APT package cache] ***************************************************************************************************************************************************
changed: [144.124.251.140]

TASK [opensearch : Configure vm.max_map_count] *************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install OpenSearch] *********************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Install OpenSearch Dashboards] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Configure OpenSearch] *******************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Create JVM options directory] ***********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Configure OpenSearch JVM heap] **********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Configure OpenSearch Dashboards] ********************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Enable and start OpenSearch] ************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Apply pending OpenSearch configuration changes] *****************************************************************************************************************************

TASK [opensearch : Wait for OpenSearch API] ****************************************************************************************************************************************************
ok: [144.124.251.140]

TASK [opensearch : Enable and start OpenSearch Dashboards] *************************************************************************************************************************************
ok: [144.124.251.140]

PLAY RECAP *************************************************************************************************************************************************************************************
144.124.251.140            : ok=18   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Единственный changed -- это `apt update`, так что всё нормально.

# Заключение

В ходе выполнения данной лабораторной работы были получены навыки работы с ansible, написания собственных ролей и развёртывния их. 

Лично для меня Ansible и ЛР в целом показались очень интересными и полезными, но единственное что не понрамилось -- то что пришлось тратиться (в моём случае ещё и дважды) на аренду серверов, ещё и не маленькой мощности. Но в остальном всё хорошо.