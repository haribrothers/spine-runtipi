# Spine for Runtipi

A [Runtipi](https://runtipi.io) app store with one app: [Spine](https://github.com/haribrothers/spine), self-hosted business management for small service and project businesses (enquiry → quote → job → invoice → payment → books).

It runs the same four containers as Spine's own install: the database, the API, the web app and Spine's proxy, from Spine's published images. Runtipi generates the secrets, and routes Spine's local subdomain (for example `https://spine.tipi.lan`) or your domain to it.

## Add the store

In Runtipi: **Settings › App Stores › Add App Store**, and paste:

```
https://github.com/haribrothers/spine-runtipi
```

Spine then appears in the app store.

## Install Spine

1. Open **Spine** in the app store and **Install**.
2. **Address for links:** the address you'll open Spine at, with no slash at the end. Spine writes it into emails, PDFs and invitations, and accepts sign-in there.
   - With Runtipi's local subdomain: turn on **expose locally**, and enter `https://spine.<your local domain>`, for example `https://spine.tipi.lan`.
   - With your own domain exposed through Runtipi: `https://spine.example.com`.
   - On the port only: `http://<the server's IP>:7746`.
3. Optional: your email server (to send quotes, invoices and invitations) and a backup passphrase.
4. Install. The first start takes a minute or two while the images download and the database is set up.

Open the address and create your account: the first person to sign up owns the business, and Spine is invite-only from then on.

**Good to know:**

- Spine also opens at `http://<the server's IP>:7746`. With an `https://` address for links, sign in through that address: over plain HTTP the browser won't keep Spine's secure sign-in cookie.
- Runtipi's local certificate is its own, so each device must trust it (download it from Runtipi's settings, under **Local domain**). The Spine field app on Android doesn't accept certificates you install yourself: for the field app, use your own domain or [Tailscale](https://spine.h4loom.com/docs/self-hosting/https-and-domains/#https-without-a-domain-tailscale).
- Your data is in Runtipi's app data folder for Spine, under `data/` (`postgres`, `files`, `snapshots`). Runtipi's own backups include it.

## Move an existing Spine install into Runtipi

If Spine already runs on this machine from `install.sh`, move its data with a backup:

1. In the old install's folder, back up and stop it (`down` without `-v` keeps its data):

   ```sh
   ./backup.sh
   docker compose --profile minimal down
   ```

   The backup is in `backups/`, e.g. `backups/spine-backup-20261116-200000.tar.gz` (`.tar.gz.enc` when encrypted).

2. Install Spine in Runtipi as above, and wait until it's running. Don't create an account.

3. Restore the backup into it. This stops Runtipi's Spine API and web app, runs Spine's restore once with the same settings and data folders, and starts them again:

   ```sh
   project=$(docker ps --format '{{.Label "com.docker.compose.project"}}' --filter label=com.docker.compose.service=api --filter label=runtipi.managed=true | grep '^spine' | head -n 1)
   api=$(docker ps -q --filter label=com.docker.compose.project=$project --filter label=com.docker.compose.service=api)
   web=$(docker ps -q --filter label=com.docker.compose.project=$project --filter label=com.docker.compose.service=web)
   image=$(docker inspect -f '{{.Config.Image}}' $api)
   network=$(docker inspect -f '{{range $name, $_ := .NetworkSettings.Networks}}{{$name}}{{end}}' $api)
   docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' $api > /tmp/spine-restore.env
   docker stop $web $api
   docker run --rm -i --network $network --env-file /tmp/spine-restore.env --volumes-from $api \
     $image node apps/api/dist/scripts/restore.js --archive - --replace --new-secret < /path/to/spine-backup-<date>.tar.gz
   rm /tmp/spine-restore.env
   docker start $api $web
   ```

   For an encrypted backup (`.tar.gz.enc`), add `-e BACKUP_PASSPHRASE='<its passphrase>'` to the `docker run` line.

   `--new-secret` is there because Runtipi generated a new secret: everyone signs in again, and AI keys and other secrets Spine stored need entering again.

4. Open Spine and sign in with your existing account. When it's all there, delete the old install's data: in its folder, `docker compose --profile minimal down -v`.

## Upgrading

When a new Spine release is out, this store is updated to it: **Settings › Update App Stores** in Runtipi, then update Spine. Spine takes a database snapshot before applying a new version's migrations.
