# Лабораторная работа: Основы Ansible в DevOps

---

## Ход работы

## 1. Установка Ansible на управляющую машину (Linux/WSL)

### Шаг 1.1: Обновление пакетов системы
```bash
sudo apt update
sudo apt upgrade -y
```
<img width="974" height="699" alt="image" src="https://github.com/user-attachments/assets/f6c860df-698d-4a8f-b409-6c92aa24d2e2" />

### Шаг 1.2: Установили Python и pip, проверили версию

<img width="974" height="625" alt="image" src="https://github.com/user-attachments/assets/993717af-294f-4db5-9d84-348cc0f11ff6" />

### Шаг 1.3: Установили Ansible и проверили версию
```bash
sudo apt install -y ansible
ansible --version
```

<img width="988" height="151" alt="image" src="https://github.com/user-attachments/assets/b4699555-2e29-420e-babe-9a1992089e88" />

Ожидаемый вывод:
```
ansible [core 2.14.x]
  config file = None
  configured module search path = ['/home/user/.ansible/plugins/modules']
  ...
```

---

## 2. Подготовка SSH ключей для управляемых машин

SSH ключи используются для аутентификации Ansible на управляемых хостах без ввода пароля.

### Шаг 2.1: Генерация SSH ключевой пары на управляющей машине
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key -N ""
```

### Шаг 2.2: Установка прав доступа на приватный ключ
```bash
chmod 600 ~/.ssh/ansible_key
chmod 644 ~/.ssh/ansible_key.pub
```
<img width="974" height="498" alt="image" src="https://github.com/user-attachments/assets/b593bbb8-434c-444b-b224-4a0da513a9f1" />

---

## 3. Запуск управляемого контейнера в Docker

### Шаг 3.1: Сборка и запуск контейнера
```bash
# Перейдите в директорию с docker-compose.yml
cd /path/to/project

# Сборка образа
docker-compose build

# Запуск контейнера в фоновом режиме
docker-compose up -d
```
<img width="974" height="665" alt="image" src="https://github.com/user-attachments/assets/cdb6b7a5-982b-419d-9dad-0c99c020b9e2" />

### Шаг 3.4: Проверка запущенного контейнера
```bash
docker-compose ps
```
<img width="974" height="163" alt="image" src="https://github.com/user-attachments/assets/ae789d40-2749-4a1c-b2db-fc2cba9665fe" />

Ожидаемый вывод:
```
NAME                    COMMAND               STATUS      PORTS
ansible-managed-host    "/usr/sbin/sshd -D"   Up 1 min    0.0.0.0:2222->22/tcp
```

## 4. Проверка SSH подключения к контейнеру

### Шаг 4.1: Проверка SSH подключения
```bash
ssh -i ~/.ssh/ansible_key -p 2222 ansible@localhost
```

Ожидаемый результат: вы должны попасть в bash контейнера без ввода пароля.
<img width="974" height="671" alt="image" src="https://github.com/user-attachments/assets/7ad6f338-fff8-4425-92e8-58e3b1eb9888" />

Выход из контейнера:
```bash
exit
```

---

## 5. Проверка инвентарного файла Ansible (inventory)

```bash
ansible-inventory -i inventory.ini --list
```
<img width="974" height="575" alt="image" src="https://github.com/user-attachments/assets/0a75eb90-9b65-405a-a21b-d7cb72f140e2" />

Ожидаемый вывод (JSON формат):
```json
{
    "_meta": {
        "hostvars": {
            "managed1": {
                "ansible_host": "localhost",
                "ansible_port": "2222",
                ...
            }
        }
    },
    "all": {...},
    "managed_hosts": {...},
    "ungrouped": {}
}
```

---

## 6. Проверка подключения Ansible к управляемому хосту

### Шаг 6.1: Тест ping
```bash
ansible -i inventory.ini managed_hosts -m ping
```
<img width="974" height="136" alt="image" src="https://github.com/user-attachments/assets/91586325-af91-4587-bf76-5beb59801968" />

Ожидаемый вывод:
```
managed1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Шаг 6.2: Сбор информации о системе (facts)
```bash
ansible -i inventory.ini managed1 -m setup
```
<img width="1024" height="642" alt="image" src="https://github.com/user-attachments/assets/63861260-ea86-4c89-88b4-4cf5ff667ccf" />

Вывело всю информацию о системе управляемого хоста.

### Шаг 6.3: Выполнение простой команды
```bash
ansible -i inventory.ini managed1 -m command -a "uname -a"
```
<img width="1107" height="60" alt="image" src="https://github.com/user-attachments/assets/03cbbc65-87fe-4ec0-940d-d77359fb5901" />

Ожидаемый вывод:
```
managed1 | CHANGED | rc=0 >>
Linux f3a4c8b0c4a2 5.15.0-92-generic #102-Ubuntu SMP Thu Jan 9 10:54:01 UTC 2025 x86_64 GNU/Linux
```

---

## 7. Создание и запуск Ansible Playbook

### Шаг 7.1: Запуск playbook
```bash
# Запуск playbook
ansible-playbook -i inventory.ini playbook.yml
```

### Шаг 7.2: Вывод playbook

<img width="974" height="563" alt="image" src="https://github.com/user-attachments/assets/a0fd54a7-5466-4bcf-ae43-7a703987ed0b" />

```
PLAY [managed_hosts] ************************************************************

TASK [Gathering Facts] **********************************************************
ok: [managed1]

TASK [Update package list] ******************************************************
changed: [managed1]

TASK [Install required packages] ************************************************
changed: [managed1]

TASK [Create test directory] ****************************************************
changed: [managed1]

TASK [Create test file with content] ********************************************
changed: [managed1]

TASK [Display file content] *****************************************************
ok: [managed1] => {
    "msg": "File content from managed host: Hello from Ansible!\nThis is a test file created by Ansible playbook."
}

TASK [Get system information] ***************************************************
ok: [managed1] => {
    "msg": "System: Linux, Hostname: f3a4c8b0c4a2, Uptime: 12 min"
}

PLAY RECAP ************************************************************
managed1 : ok=7 changed=4 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

---

## 8. Задания для выполнения

### Задание 1: Базовое подключение
1. Установили Ansible на нашей машине
2. Сгенерировали SSH ключевую пару
3. Создали инвентарный файл `inventory.ini`
4. Проверили подключение командой `ansible-inventory --list`
5. Выполнили ping к управляемому хосту

---

### Задание 2: Базовые ad-hoc команды
1. Получите информацию о ядрах CPU управляемого хоста:
   ```bash
   ansible -i inventory.ini managed1 -m setup -a "filter=ansible_processor_cores"
   ```

2. Проверьте свободное место на диске:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "df -h"
   ```

3. Получите список всех пользователей:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "cat /etc/passwd"
   ```

4. Измените временную зону хоста на UTC:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "timedatectl set-timezone UTC"
   ```
<img width="974" height="642" alt="image" src="https://github.com/user-attachments/assets/dab396a5-79ab-4812-9b20-31d6e7358c89" />

---

### Задание 3: Работа с файлами
1. Создали и запустиои новый playbook `task3_files.yml`:
    ```bash
   ansible-playbook -i inventory.ini task3_files.yml
   ```

<img width="974" height="552" alt="image" src="https://github.com/user-attachments/assets/d48f2de1-b951-42cc-9785-59e25924b2e1" />

---

## 10. Вывод

В ходе выполнения лабораторной работы установили Ansible на управляющую машину в системе linux, создали SSH ключи для управляемых машин,собрали и запустили управляемый контейнер в Docker, проверили SSH подключение к контейнеру, создали инвентарный файл Ansible (inventory), проверили подключение Ansible к управляемому хосту, создали и запустили Ansible Playbook, а также выполнили базовые ad-hoc команды.
---
