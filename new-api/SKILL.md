---
name: new-api
description:
license: Apache 2.0
---
To create a new api <endpoint> accessible as /api/my/v1/<endpoint>
- use python to implement it
- choose <endpoint> to be using only letters, numbers and '-' and starting with a letter
- be <package> the same as <endpoint> replacing '_' with '-'
-  reate a folder `packages/v1/<endpoint>`
- write a file `packages/v1/<endpoint>/__main__.py` with the content:

```
#--kind python:default
#--web true
## start params
## end params
import <package>
def main(args):
  try:
    return { "body": <package>.main(args) }
  except Exception as e:
    return {
      "body": str(e),
      "statusCode": 500
    }
```

- write a file `packages/v1/<endpoint>/<package>.py` with the content:

```
def main(args):
    inp = args.get("input", "<package>")
    out = inp
    return out
```
