# Тестовое задание: Развёртывание и настройка Foreman + Ansible

## 1. Архитектура тестового стенда
Стенд развёрнут локально в среде виртуализации VirtualBox на базе ОС **Debian 13 (Trixie, x86_64)** и состоит из двух виртуальных машин, объединённых во внутреннюю изолированную сеть (`192.168.100.0/24`):

* **VM1 (`foreman.lab.local` / `192.168.100.10`):** Сервер управления Foreman 5.0 с установленными плагинами `foreman_plugin_ansible` и `foreman_proxy_plugin_ansible`.
* **VM2 (`client1.lab.local` / `192.168.100.20`):** Управляемый хост (Managed Host), зарегистрированный в Foreman через механизм *Global Registration* и привязанный к созданной группе узлов.

---

## 2. Выполненные шаги и описание конфигурации

### 2.1. Интеграция с Ansible и роль запрета входа `root`
На сервере Foreman создана роль `Запрещаем вход под root по SSH` в директории `/etc/ansible/roles/disable_root_ssh/tasks/main.yml` и импортирована в веб-интерфейс 

**Как работает роль:**
1. Модуль `lineinfile` открывает `/etc/ssh/sshd_config`, находит строку `PermitRootLogin` и заменяет её на `PermitRootLogin no`.
2. Модуль `service` перезапускает службу `ssh`, чтобы новые параметры сразу вступили в силу.
<img width="1918" height="911" alt="Ansible_role" src="https://github.com/user-attachments/assets/fb70fc2d-a789-447c-8012-65ea073a01b3" />


### 2.2. Provisioning Template с настройкой NTP
На базе стандартного шаблона `Preseed default` создан пользовательский шаблон `Preseed default Debian13` (`templates/preseed default Debian13.erb`)

В блок постобработки `d-i preseed/late_command` добавлена автоматическая установка и настройка службы синхронизации времени `chrony`:
```bash
in-target apt-get update && in-target apt-get install -y chrony && in-target bash -c 'echo "pool 0.ru.pool.ntp.org iburst" >> /etc/chrony/chrony.conf' && in-target systemctl enable chrony
```
<img width="1908" height="908" alt="Preseed default" src="https://github.com/user-attachments/assets/445d6c66-e736-4c40-978a-65da8dc581cd" />

### 2.3. Настройка Host Group и регистрация Managed Host
1. Создана группа узлов (**Host Group**), объединяющая ОС Debian 13, шаблон развёртывания с NTP и Ansible-роль `disable_root_ssh` 


2. Узел `client1.lab.local` зарегистрирован в Foreman с привязкой к созданной группе узлов *(скриншот: `screenshots/Client1.png`)*
.

3. Через веб-интерфейс Foreman запущено применение унаследованных Ansible-ролей (*Run Ansible roles*). Задача завершилась со статусом **Succeeded** *(скриншот: `screenshots/Client 1 Ansible.png`)*.


4. Проверка на стороне клиента (`grep PermitRootLogin /etc/ssh/sshd_config`) подтвердила, что вход под `root` отключён (`PermitRootLogin no`)
<img width="817" height="43" alt="root check" src="https://github.com/user-attachments/assets/ea437a92-9373-426a-b939-50b5f1c531f9" />

---

## 3. Сравнение подходов: Foreman vs ручной стек (Docker + Nginx + Samba)
.

| Критерий | Ручной стек (Docker + Nginx + Samba) | Foreman (+ Ansible) |
| :--- | :--- | :--- |
| **Суть подхода** | Файловая раздача: сервер отдаёт образы, скрипты и конфиги по HTTP/SMB. | Оркестратор полного цикла: от установки ОС (Provisioning) до контроля конфигураций. |
| **Учёт и инвентаризация** | Ведётся вручную (таблицы, заметки). Нет единой актуальной базы хостов. | **Single Source of Truth:** единый реестр всех узлов, их железа (Facts), IP/MAC и статусов. |
| **Контроль дрейфа (Идемпотентность)** | Скрипт выполняется один раз при установке. Если конфиг потом поменяют руками, система об этом не узнает. | Интеграция с Ansible позволяет повторно приводить серверы к эталону и отслеживать изменения в отчётах. |
| **Стандартизация и группы** | При росте парка машин приходится править множество разрозненных скриптов. | **Host Groups** задают единый профиль сервера (ОС + NTP + роли Ansible) для массового тиражирования. |
| **Безопасность и аудит** | Общие сетевые папки и пароли, нет разграничения прав и истории запусков. | Ролевая модель доступа (RBAC), работа по SSH-ключам через Smart Proxy и полный журнал всех задач. |
