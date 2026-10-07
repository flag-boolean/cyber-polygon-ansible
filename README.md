# cyber-polygon-ansible
# Ansible + OpenNebula: лабораторные стенды

Автоматизация развёртывания изолированных лабораторий (например, GOAD)
в OpenNebula через Ansible.

## Архитектура

- **Хост A (Control Node)** — здесь запускается Ansible. OpenNebula не установлена.
- **Хост B (OpenNebula Frontend)** — здесь работает OpenNebula. Ansible подключается
  по SSH под пользователем `oneadmin` и выполняет все команды там.

Управление VM идёт через XML-RPC API OpenNebula (`http://127.0.0.1:2633/RPC2`)
с использованием `pyone`. Это позволяет не парсить вывод CLI и корректно
работать с сетями, шаблонами и VM.

## Требования

### Хост A (Control Node)
- Ansible 2.15+
- Коллекция `community.general`
- SSH-доступ до хоста B под пользователем `oneadmin` (беспарольный)

### Хост B (OpenNebula Frontend)
- OpenNebula установлена и работает
- `python3-pyone` (или `pyone` в системном Python)
- CLI-утилиты `onevm`, `onevnet`, `onetemplate` в `$PATH`
- VXLAN-сеть с физическим интерфейсом `br0` (проверьте `ip a`)

## Подготовка

### 1. Установить Ansible на хосте A

```bash
sudo apt update
sudo apt install -y ansible
ansible-galaxy collection install community.general
```

### 2. Установить pyone на хосте B

```bash
sudo apt install -y python3-pyone
python3 -c "import pyone; print('pyone OK')"
```

### 3. Настроить SSH-ключи

```bash
# На хосте A
ssh-keygen -t ed25519 -f ~/.ssh/oneadmin_key -N ""
ssh-copy-id -i ~/.ssh/oneadmin_key.pub oneadmin@<IP_Хоста_B>
```

### 4. Создать Vault с паролем OpenNebula

```bash
ansible-vault create secrets.yml
# Внутри:
# vault_one_api_password: "ваш_пароль_oneadmin"
```

## Использование

### Создать ВСЕ лаборатории из `vars/labs.yml`

```bash
ansible-playbook -i inventory.ini create_labs.yml --ask-vault-pass
```

### Создать одну лабораторию

```bash
ansible-playbook -i inventory.ini create_lab.yml \
  -e "lab_name=goad-lab1" --ask-vault-pass
```

### Пересоздать одну лабораторию (удалить + создать заново)

```bash
ansible-playbook -i inventory.ini recreate_lab.yml \
  -e "lab_name=goad-lab1" --ask-vault-pass
```

### Удалить одну лабораторию

```bash
ansible-playbook -i inventory.ini destroy_lab.yml \
  -e "lab_name=goad-lab1" --ask-vault-pass
```

### Удалить ВСЕ лаборатории

```bash
ansible-playbook -i inventory.ini destroy_lab.yml \
  -e "lab_name=all" --ask-vault-pass

# или просто без параметра — default('all')
ansible-playbook -i inventory.ini destroy_lab.yml --ask-vault-pass
```

## Как добавить новую лабораторию

Отредактируйте `vars/labs.yml`, добавив запись в список `labs`.
Все playbooks автоматически её подхватят.

## Особенности реализации

- **VXLAN-сеть с пулом IP** создаётся через CLI `onevnet create`, потому что
  модуль `community.general.one_vnet` некорректно передаёт блок `AR`
  через XML-RPC.
- **VM создаются через CLI `onetemplate instantiate`**, потому что модуль
  `community.general.one_vm` не передаёт `NETWORK_UNAME` в атрибут `NIC`,
  что приводит к ошибке `User 0 does not own a network`.
- **ВМ ищутся по имени, а не по метке `LABELS`**, потому что фильтр
  `onevm list --filter "LABELS=..."` в OpenNebula не работает с вложенным
  `USER_TEMPLATE/LABELS`. Имена вида `goad-labN-role-NNN` — надёжный
  идентификатор.
