---
name: use-secret
description: how to use a secret configured in the environment
license: Apache 2.0
---

Secrets are stored in `.env` for development and `.env.production` for production
To use a secret like <MY_API_KEY> in a backend function: `/api/my/<package>/<action>`
you have to edit the file `packages/<package>/<action>/__main__.py`
adding a line after `## start params` in format:

```
#--param <MY_API_KEY> "$<MY_API_KEY>"
```

the secret will be then available and you can read in file

`packages/<package>/<action>/<action>.py`

with:

```
<MY_API_KEY> = args.get("<MY_API_KEY>", os.getenv("<MY_API_KEY>"))
````

the fallback `os.getenv` is required for the tests