1. Dockerfile — для управляемого хоста

```yml
FROM ubuntu:22.04

# Установка необходимых пакетов
RUN apt-get update && apt-get install -y \
    openssh-server \
    openssh-client \
    python3 \
    python3-pip \
    sudo \
    curl \
    wget \
    git \
    nano \
    htop \
    && rm -rf /var/lib/apt/lists/*

# Создание пользователя ansible
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Настройка SSH
RUN mkdir -p /run/sshd && \
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Создание .ssh директории для пользователя ansible
RUN mkdir -p /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh

# Запуск SSH сервера в foreground режиме
CMD ["/usr/sbin/sshd", "-D"]
```

2. docker-compose.yml — конфигурация контейнера
```yml
version: '3.8'

services:
  managed-host:
    container_name: ansible-managed-host
    build: .
    ports:
      - "2222:22"
    environment:
      - TERM=xterm-256color
    networks:
      - ansible-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "service", "ssh", "status"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  ansible-net:
    driver: bridge
```

3. inventory.ini — инвентарный файл

```yml
# Инвентарный файл для Ansible
# Описывает управляемые хосты

[managed_hosts]
managed1 ansible_host=localhost ansible_port=2222 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/ansible_key ansible_python_interpreter=/usr/bin/python3

[all:vars]
# Отключаем проверку SSH ключа хоста при первом подключении
ansible_ssh_common_args=-o StrictHostKeyChecking=no
# Таймаут подключения (секунды)
ansible_connection_timeout=10
```

4. playbook.yml — основной playbook

```yml
---
- name: Демонстрационный playbook для Ansible
  hosts: managed_hosts
  gather_facts: yes
  
  tasks:
    - name: Обновление списка пакетов
      apt:
        update_cache: yes
        cache_valid_time: 3600
      become: yes

    - name: Установка требуемых пакетов
      apt:
        name:
          - curl
          - wget
          - git
          - nano
        state: present
      become: yes

    - name: Создание тестовой директории
      file:
        path: /tmp/ansible_test
        state: directory
        mode: '0755'

    - name: Создание тестового файла с содержимым
      copy:
        content: |
          Hello from Ansible!
          This is a test file created by Ansible playbook.
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Date: {{ ansible_date_time.iso8601 }}
        dest: /tmp/ansible_test/test_file.txt
        mode: '0644'

    - name: Вывод содержимого файла
      command: cat /tmp/ansible_test/test_file.txt
      register: file_content

    - name: Отображение содержимого файла
      debug:
        msg: "{{ file_content.stdout }}"

    - name: Сбор информации о системе
      debug:
        msg: |
          System: {{ ansible_system }}
          Hostname: {{ ansible_hostname }}
          Uptime: {{ ansible_uptime_seconds }} seconds
          CPU cores: {{ ansible_processor_cores }}
          Total memory: {{ ansible_memtotal_mb }} MB
```


5. Playbook работа с файлами
```
---
- name: Работа с файлами
  hosts: managed_hosts
  
  tasks:
    - name: Создание нескольких директорий
      file:
        path: /tmp/{{ item }}
        state: directory
        mode: '0755'
      loop:
        - test_dir1
        - test_dir2
        - test_dir3

    - name: Создание файлов в директориях
      copy:
        content: "Это файл {{ item }}\nСоздан Ansible playbook\n"
        dest: /tmp/{{ item }}/content.txt
      loop:
        - test_dir1
        - test_dir2
        - test_dir3

    - name: Вывод содержимого файлов
      command: cat /tmp/{{ item }}/content.txt
      loop:
        - test_dir1
        - test_dir2
        - test_dir3
      register: file_content

    - name: Отображение содержимого
      debug:
        msg: "Файл {{ item.item }}: {{ item.stdout }}"
      loop: "{{ file_content.results }}"
```
