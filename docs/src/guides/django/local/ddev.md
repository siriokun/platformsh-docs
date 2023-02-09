---
title: DDEV
weight: -110
layout: single
description: |
    Set up an environment with Platform.sh's recommended local development tool, DDEV.
sectionBefore: Integrated environments
---

{{% ddev/definition %}}

{{% guides/local-requirements %}}
- DDEV installed on your computer.
  
  {{% ddev/requirements %}}

  {{% ddev/install %}}

{{% guides/django/local-assumptions %}}

## Set up DDEV

1.  Create a new environment off of production.

    ```bash
    platform branch new-feature main
    ```

    If you're using a [source integration](../../../integrations/source/_index.md),
    open a merge/pull request.

2.  Add DDEV configuration to the project.

    ```bash
    ddev config --auto
    ```

3.  Add a Python alias package to the configuration.

    ```bash
    ddev config --webimage-extra-packages python-is-python3
    ```

4.  Add an API token.

    {{% ddev/token %}}

5.  Connect DDEV to your project and the `new-feature` environment.

    {{% ddev/connect %}}

6.  Update the DDEV PHP version.

    Python support in DDEV and the Platform.sh integration is in active development.
    At this time, the only officially supported runtime is PHP.
    With a few changes, the generated configuration can be modified to run local Django environments.

    A `.ddev` directory has been created in the repository containing DDEV configuration. 

    In the `.ddev/config.platformsh.yaml` file, update the `php_version` attribute to a supported version, like `8.2`.

    ```yaml {location=".ddev/config.platformsh.yaml"}
    # Leaving the generated line as is results in an error.
    # php_version: python:3.10
    php_version: 8.2
    ```

7.  Update the DDEV `post-start` hooks.

    The generated configuration contains a `hooks.post-start` attribute that contains Django's `hooks.build` and `hooks.deploy`. 
    Add another item to the end of that array with the start command defined in `.platform.app.yaml`:

    {{< codetabs >}}
+++
title=Pip
+++
```yaml {location=".ddev/docker-compose.django.yaml"}
hooks:
    post-start:
        ...
        # Platform.sh start command
        - exec: python manage.py runserver 0.0.0.0:8000
```
<--->
+++
title=Pipenv
+++
```yaml {location=".ddev/docker-compose.django.yaml"}
hooks:
    post-start:
        ...
        # Platform.sh start command
        - exec: pipenv run python manage.py runserver 0.0.0.0:8000
```
<--->
+++
title=Poetry
+++
```yaml {location=".ddev/docker-compose.django.yaml"}
hooks:
    post-start:
        ...
        # Platform.sh start command
        - exec: poetry run python manage.py runserver 0.0.0.0:8000
```
    {{< /codetabs >}}

8.  Create a custom Docker Compose file to define the port exposed for the container you're building, `web`:

    ```yaml {location=".ddev/docker-compose.django.yaml"}
    version: "3.6"
    services:
        web:
            expose:
                - 8000
            environment:
                - HTTP_EXPOSE=80:8000
                - HTTPS_EXPOSE=443:8000
            healthcheck:
                test: "true"
    ```

9.  Create a custom `Dockerfile` to install Python into the container.

    ```{location=".ddev/web-build/Dockerfile.python"}
    RUN apt-get install -y python3.10 python3-pip
    ```

10. Update your allowed hosts to include the expected DDEV domain suffix:

    ```py {location="APP_NAME/settings.py"}
    ALLOWED_HOSTS = [
        'localhost',
        '127.0.0.1',
        '.platformsh.site',
        '.ddev.site'
    ]
    ```

11. Update your databases configuration to include environment variables provided by DDEV:

    ```py {location="APP_NAME/settings.py"}
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': os.environ.get('POSTGRES_NAME'),
            'USER': os.environ.get('POSTGRES_USER'),
            'PASSWORD': os.environ.get('POSTGRES_PASSWORD'),
            'HOST': 'db',
            'PORT': 5432,
        }
    }
    ```        

12. Start DDEV.

    Build and start up Django for the first time. 

    ```bash
    ddev start
    ```

    If you have another process running, you may get an error such as the following:

    ```bash
    Failed to start django: Unable to listen on required ports, port 80 is already in use
    ```

    If you do, either cancel the running process on port `80` or do both of the following:

    - Edit `router_http_port` in `.ddev/config.yaml` to another port such as `8080`.
    - Edit the `services.web.environment` variable `HTTP_EXPOSE` in `ddev/docker-compose.django.yaml` to `HTTP_EXPOSE=8080:8000`.

13. Pull data from the environment.

    Exit the currently running process (`CTRL+C`)
    and then run the following command to retrieve data from the current Platform.sh environment:

    ```bash
    ddev pull platform
    ```

14. Restart DDEV

    ```bash
    ddev restart
    ```

    You now have a local development environment that's in sync with the `new-feature` environment on Platform.sh.

15. When you finish your work, shut down DDEV.

    ```bash
    ddev stop
    ```

## Next steps

You can now use your local environment to develop changes for review on Platform.sh environments.
The following example show how you can take advantage of that.

### Onboard collaborators

It's essential for every developer on your team to have a local development environment to work on. 
Place the DDEV configuration into a script to ensure everyone has this.
You can merge this change into production. 

1.  Create a new environment called `local-config`.

2.  Create an executable script to set up a local environment for a new Platform.sh environment. 

    ```bash
    touch init-local.sh && chmod +x init-local.sh
    ```

    Fill it with the following example, depending on your package manger:

    {{< codetabs >}}
+++
title=Pip
+++
{{< readFile file="snippets/guides/django/local-pip.sh" highlight="yaml" location="init-local.sh" >}}
<--->
+++
title=Pipenv
+++
{{< readFile file="snippets/guides/django/local-pipenv.sh" highlight="yaml" location="init-local.sh" >}}
<--->
+++
title=Poetry
+++
{{< readFile file="snippets/guides/django/local-poetry.sh" highlight="yaml" location="init-local.sh" >}}
    {{< /codetabs >}}

3.  Commit and push the revisions by running the following command:

    ```bash
    git add . && git commit -m "Add local configuration" && git push platform local-config
    ```

4.  Merge the change into production by running the following command:

    ```bash
    platform merge local-config
    ```

Once the script is merged into production,
any user can then set up their local environment by running the following commands:

```bash
$ platform get {{< variable "PROJECT_ID" >}}
$ cd {{< variable "PROJECT_NAME" >}}
$ ./init-local.sh {{< variable "PROJECT_ID" >}} another-new-feature main
```

### Sanitize data

It's often a compliance requirement to ensure that only a minimal subset of developers within an organization
have access to production data during their work.
By default, your production data is automatically cloned into _every_ child environment.

You can customize your deployments to include a script that sanitizes the data within every non-production environment. 

1.  Create a new environment called `sanitize-non-prod`.

2.  Follow the example on how to [sanitize PostgreSQL with Django](../../../development/sanitize-db/postgresql.md).
    This adds a sanitization script to your deploy hook that runs on all non-production environments.

3.  Commit and push the revisions by running the following command:

    ```bash
    git add . && git commit -m "Add data sanitization" && git push platform sanitize-non-prod
    ```

4.  Merge the change into production by running the following command:

    ```bash
    platform merge sanitize-non-prod
    ```

Once the script is merged into production, every non-production environment created on Platform.sh
and all local environments contain sanitized data free of your users' personally identifiable information (PII).
