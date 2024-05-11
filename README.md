# Symfony + PHPStorm + Docker > Dev Environment + Project Setup

We will create a Symfony project in PHPStorm, and have a working Database, Debug and Testing without PHP installed on our system.

This is just a quick summary of what you need to do. It will not cover why you're doing it or how to use it.

## Notes and Errors.

! Route names will contain `index.php`:
- Your home `'/'` route will look like this http://localhost:8080/index.php
- `'/login'` route will look like this http://localhost:8080/index.php/login
- Any input on how to fix this is appreciated.

! Windows+WSL users:
- Use WSL, you will suffer severe performance issues if you don't.
- WSL installation is not covered in this guide.
- After creating a fresh project using docker, you might not be the directory owner. You'll know this if PHPStorm is throwing errors about not being able to save configuration.
- Check file ownership by running `ls -la` in your WLS terminal in project root. Is should not say `root root` next to files.
- Change dir ownership (recursively) by running `sudo chown -R USER: .` in project root.
- run `id`.
- Find your users and groups `uid` and `gid` numbers
- Add `user: "{uid}:{gid}"` to your `compose.yaml`
  ```sh
  web:
    user: "1000:1000"
    build: .
  ```
- This is a quick fix so you don't need to update your file owner after creating anything with the `make:` command. Please contribute a better solution.


! Linux users:
- In compose file `XDEBUG_CONFIG: client_host=host.docker.internal` might not work for you.
- Try instead ` XDEBUG_CONFIG: remote_host=<hostname>` where `<hostname>` is replaced with results of `ip -4 addr show docker0 | grep -Po 'inet \K[\d.]+'` command.
- [More details](https://github.com/docker/for-linux/issues/264)


## Initial Setup

- Install PHPStorm
- Install Docker
- Install browser debugging extensions from [here](https://www.jetbrains.com/help/phpstorm/2024.1/browser-debugging-extensions.html)
- In PHPStorm go to `Settings > Plugins > Marketplace` and install `Symfony Support`.
- `Settings > Editor > Code Style > PHP > Set from... > Symfony2`
- (opt) Make sure that we have a Docker connection set up `Settings > Build, Execution, Deployment > Docker`

![](/images/d1.png)

## Create New Symfony project

- Open a terminal. Navigate you your project parent directory.
- Copy command below. Replace PROJECT_FOLDER with project name, and execute.
```sh
docker run --rm --interactive --tty --volume $PWD:/app \
    composer create-project symfony/skeleton --no-progress --no-interaction --ansi --ignore-platform-reqs PROJECT_FOLDER
```
- Open newly created folder in PHPStorm.


## Create docker files

A note on docker compose file name from [here](https://docs.docker.com/compose/compose-application-model/) :
> The default path for a Compose file is compose.yaml (preferred) or compose.yml that is placed in the working directory. Compose also supports docker-compose.yaml and docker-compose.yml for backwards compatibility of earlier versions. If both files exist, Compose prefers the canonical compose.yaml.

- Create file `Dockerfile` in project root.
- [Check here](https://hub.docker.com/r/phpstorm/php-apache/tags) for newer/different PHP version.

```Docker
FROM phpstorm/php-apache:8.2-xdebug3.2

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public

RUN apt-get update && apt-get -y install libzip-dev libicu-dev
RUN docker-php-ext-install pdo pdo_mysql zip mysqli
RUN docker-php-ext-enable pdo pdo_mysql zip mysqli

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

COPY . /var/www/html
```

- Create file `compose.yaml` in project root.

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    links:
      - db
    volumes:
      - ./:/var/www/html
    environment:
      XDEBUG_MODE: develop,debug
      XDEBUG_CONFIG: client_host=host.docker.internal
  db:
    image: mysql:latest
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: symfony_app
```
- Press the “Play” button in the `compose.yaml` file to get our services started.

![](images/d2.png)

### Go to http://localhost:8080/index.php
### You now have a working Symfony server.

## Debug

You can autogenerate a server without having to add this by trying to run a debug session. Then you'll only have to fill in the last step with root dir.
- `Settings > PHP > Servers`
- Press `Add (insert)`
- host: `localhost`
- port: `8080`
- Use path mappings: Checked
- Map project root to the `/var/www/html` directory inside the container

![](images/d3.png)

### You can now use Xdebug

## Auto Reformat on save

- `Settings > Tools > Actions on Save`
- Check first 3 [Reformat code, Optimize imports, Rearrange code]

![](images/s1.png)

## Running PHP/Composer/Symfony commands

Open docker container terminal through PHPStorm. You will run all your commands here.
- `Services > Docker > Docker-compose: [your project] > web > [web-container] > Terminal`

![](images/d4.png)
![](images/d5.png)

### Alternative

Open docker container terminal using commands. We could use `docker exec -it [container_name] bash`, but container name will change as you make more and more `web` containers.

In stead, use:
```sh
docker compose exec -it web bash
```
It will always connect to the web container from your compose file.

Use `exit` or `Ctrl-D` to return to your terminal.

You can also execute command without going into bash like this:
```sh
docker compose exec web php bin/console make:test
```

### You can now start installing Symfony bundles with composer

## Symfony bundles

- Install packages you need.
- Don't include docker from orm-pack.
- You can copy and run multiple lines if they end with `;`

```sh
# Basic Bundles
composer require form validator symfony/orm-pack;
composer require --dev symfony/profiler-pack symfony/maker-bundle;
# Medium Bundles
composer require symfony/security-bundle symfony/http-client;
composer require --dev symfony/test-pack orm-fixtures;
# Advanced Bundles
composer require symfony/cache symfony/messenger symfony/workflow symfony/scheduler;
```

## Database

- Open `.env`
- Set up your `DATABASE_URL`
```sh
###> doctrine/doctrine-bundle ###
...
# DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
DATABASE_URL="mysql://root:root@db:3306/symfony_app?serverVersion=8.0.32&charset=utf8mb4"
# DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
# DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
###< doctrine/doctrine-bundle ###
```

Connecting to database through PHPStorm

- `Database > + (New) > Data Source > MySQL`
- Input DB data set in `compose.yaml`
- Click `Test Connection`. Save.

![](images/db1.png)
![](images/db2.png)

## Useful Commands

```sh
# Make
php bin/console make:controller
php bin/console make:form
php bin/console make:command
php bin/console make:listener # make:subscriber command is deprecated

# User
php bin/console make:user
php bin/console make:auth
php bin/console make:voter
php bin/console make:registration-form
php bin/console security:hash-password # Hash pass for fixture.

# ORM
php bin/console make:entity
php bin/console make:migration
php bin/console doctrine:migrations:migrate

# Fixtures
php bin/console make:fixtures
php bin/console doctrine:fixtures:load

# Other
php bin/console debug:router # Quickly see all route names
php bin/console messenger:consume async
php bin/console scheduler:consume

# Tests
php bin/console make:test
php bin/phpunit # Run all tests
php bin/phpunit tests/Form # Run all tests in Form/ directory
php bin/phpunit tests/Form/UserTypeTest.php # run tests for the UserType class

```

## Tests

Set up docker as CLI interpreter to run tests through PHPStorm.
- `Settings > PHP`

![](images/c1.png)
![](images/c2.png)
![](images/c3.png)
![](images/c4.png)

### You can now run and debug tests

- Try running a test with a 'Play' button. PHPStorm should create `Test Framework` in your settings automatically.
- You can go to `Settings > PHP > Test Frameworks` to add a `Default configuration file`
- After you run a test from a file, PHPStorm will add it to your 'Run' menu. You can continue running test from file, use the Run menu, use Shortcuts, or run `php bin/phpunit` in the console.

## Add Bootstrap to Project and forms

- open `/templates/base.html.twig`
- add latest bootstrap [links](https://getbootstrap.com/)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Welcome!{% endblock %}</title>
    <link rel="icon"
          href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
    {% block stylesheets %}
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
              rel="stylesheet"
              integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH"
              crossorigin="anonymous">
    {% endblock %}

    {% block javascripts %}
        <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
                integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz"
                crossorigin="anonymous"></script>
    {% endblock %}
</head>
<body>
{% block body %}{% endblock %}
</body>
</html>
```

- open `/config/packages/twig.yaml`
- add `form_themes: ['bootstrap_5_layout.html.twig']`
```yaml
twig:
    file_name_pattern: '*.twig'
    form_themes: ['bootstrap_5_layout.html.twig']

when@test:
    twig:
        strict_variables: true
```


## Cleanup

- Delete empty `.gitignore` from all `/src/*` folders and `/migrations`