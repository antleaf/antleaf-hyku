## Local Development

The local Stack Car hostname is driven by `APP_NAME`.

This can be configured in `.env.development` which has precedence over the default `.env` file.


### Eample `.env.development` file
```env
APP_NAME=antleaf-hyku
INITIAL_ADMIN_EMAIL=support@antleaf.com
INITIAL_ADMIN_PASSWORD=$ANTLEAF_HYKU_ADMIN_PASSWORD
```

```bash
sc up
```

Then open:

```text
https://admin-antleaf-hyku.localhost.direct/
```

To check the active admin host:

```bash
sc config | grep HYKU_ADMIN_HOST
```
