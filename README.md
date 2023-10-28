# GitLab как локальное приложение с помощью Docker
Задание: развернуть локальную инстанцию GitLab с использованием Docker, настроить и проверить конвейер с использованием Git, а также настроить аутентификацию. 
Разделено на два основных раздела: один для Windows 10 с WSL, и другой для Linux (требуется Ubuntu 22.04 LTS или 23.04).


## Подготовка окружения
```
sudo apt install ssh  # Установка/проверка агента SSH
sudo apt install git  # Установка/проверка агента для работы с Git репозиториями (по необходимости)
sudo apt install docker  # Установка/проверка Docker
```

При необходимости удалите неофициальные пакеты, связанные с Docker:
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

## Создание локального проекта
Создайте папку для вашего проекта:
```
mkdir leading_engineering_schools
cd leading_engineering_schools
```

Создайте файл docker-compose.yml и добавьте следующий код:
```
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://172.17.0.1:80'
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
      - '80:80'
      - '443:443'
      - '22:22'
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
    shm_size: '256m'
  runner:
    image: 'gitlab/gitlab-runner:latest'
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
      - '$GITLAB_HOME/runner-config:/etc/gitlab-runner'
```
Важно: Если вы не используете WSL, измените строку external_url на http://127.0.0.1:80.

Запустите следующие команды для сборки и запуска контейнеров приложения:
```
export GITLAB_HOME=/home/adam/devops/
docker compose up -d
```

Подождите несколько минут, чтобы убедиться, что контейнеры успешно запущены.

### Настройка GitLab и Glab
Откройте браузер и перейдите по адресу http://localhost:80.
Используйте пользователя root. Пароль пользователя GitLab можно найти в файле /etc/gitlab/initial_root_password внутри контейнера GitLab. Чтобы просмотреть его, выполните:

```
cat $GITLAB_HOME/config/initial_root_password
```

Установите glab для работы через командную строку с GitLab:
```
sudo snap install glab
```

Затем выполните вход в GitLab:
```
glab auth login
```

Выберите опцию GitLab Self-hosted Instance, и введите следующие параметры:
```
GitLab hostname: localhost
API hostname: (оставьте пустым, нажмите Enter)
How would you like to login?
> Token
```
Получите токен доступа для GitLab по адресу http://localhost/-/profile/personal_access_tokens.

Авторизуйтесь с помощью этого токена на локальном клиенте glab:
```
glab auth login
```

Теперь вы можете подключиться к GitLab через glab.

##  Настройка SSH ключа
Установите SSH и сгенерируйте SSH ключ:
```
sudo apt install ssh
ssh-keygen -t rsa
sudo chmod 400 ~/.ssh/id_rsa.pub
```

Добавьте ваш сгенерированный SSH ключ в настройках GitLab, чтобы иметь доступ к репозиторию.

## Укажите настройки пользователя для Git:
```
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

## Настройка GitLab Runner и создание репозитория
Внутри контейнера GitLab Runner выполните команду для регистрации раннера. Зайдите в командную строку контейнера с помощью:
```
docker exec -it <runner container name> bash
```

Создайте удаленный репозиторий в GitLab, затем добавьте простой файл .gitlab-ci.yml, который создает текстовый файл file.txt в качестве артефакта.

Добавьте репозиторий в glab:
```
glab repo clone <repository_url>
cd <repository_name>
```
Отправьте файл .gitlab-ci.yml в репозиторий и убедитесь, что конвейер успешно завершается.

## Особенности работы
1. Для доступа к GitLab используйте адрес 172.17.0.1:80 изнутри контейнера, чтобы обратиться к localhost.
2. Пароль пользователя root для установленного GitLab находится в файле /etc/gitlab/initial_root_password внутри контейнера.
3. При установке GitLab Runner GitLab выдаст код для его регистрации. Выполните эту команду внутри контейнера GitLab Runner.
4. Для того чтобы GitLab узнал ваш репозиторий, добавьте содержимое файла id_rsa.pub из папки ~/.ssh/ в настройки вашего локального (http://localhost) GitLab.
