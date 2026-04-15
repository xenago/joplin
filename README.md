# joplin

Deploy a container stack for [Joplin](https://github.com/laurent22/joplin), an AGPL-licensed management system for Markdown notes.

## Architecture

Joplin is comprised of several components:

* Applications
  * Desktop
    * Fully-featured, allows editing metadata and all complex functionality
  * Mobile
    * Lacks some features
    * Not compatible with all plugins
  * Webapp
    * The mobile app can be built for the web target platform
    * Not fully supported yet, still in beta
    * Requires building from source if self-hosting
* Server
  * The **server component is optional**
    * Used mainly for syncing notes
    * Alternatively, existing **WebDAV/S3/Cloud storage can be used instead of the server component**
  * Postgres DB is deployed alongside this and used for note storage and account metadata
    * To avoid DB bloat we will use the filesystem for blob storage, like `STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir`:
  * An existing HTTPS proxy deployment is required, **Nginx** is used here
  * If you have an SMTP server for transactional mail, it can be used with Joplin, but that is omitted here
  * An ML-powered transcription engine for server can optionally be deployed alongside the server container, but that is not included here

## Resources

Documentation is a bit spread out for Joplin. It could use some updates and consolidation.

Here are the docs I used:

* Official docs (lacking)
  
  https://github.com/laurent22/joplin/tree/dev/packages/server

* Github source (best)
  
  https://github.com/laurent22/joplin/tree/dev/packages/server

* Discussion on forum regarding server docs
  
  https://discourse.joplinapp.org/t/joplin-server-documentation/24026/36

* Official doc on building webapp

  https://joplinapp.org/help/dev/BUILD/#web

* Forum discussion about webapp

  https://discourse.joplinapp.org/t/web-client-running-joplin-mobile-in-a-web-browser-with-react-native-web/38749/23

* Deploying the webapp on GitHub Pages

  https://github.com/joplin/web-app/blob/main/.github/workflows/deploy-github-pages.yml

* `.env` example

  https://raw.githubusercontent.com/laurent22/joplin/dev/.env-sample

* Compose stack example including (ML transcription components)

  https://github.com/laurent22/joplin/blob/dev/docker-compose.server.yml

* Using filesystem for attachment storage (avoids bloating postgres DB size)

  https://github.com/laurent22/joplin/tree/dev/packages/server#setting-up-storage-on-a-new-installation

## Deployment notes

The following aspects are relevant for this specific deployment:
* ML transcription is not used
* SMTP is not used
* Encryption is not used
* Nginx proxy is used
* Optional webapp is built from source into a minimal container if needed
* Optional exports can be enabled as a secondary backup option
* IPv4 is used (IPv6 users will need to make some changes according to their own setups)

### Webapp deployment notes

The webapp is not yet fully supported, but does work. It's the mobile app, just built for the web target platform instead.

I set up a [Dockerfile](joplin-webapp-Dockerfile) which builds the webapp and hosts the static files. This is based on work from these two PRs:
* https://github.com/laurent22/joplin/pull/12563
* https://github.com/joplin/web-app/pull/2

When new webapp versions are released, update the Dockerfile accordingly. Aside from any changes noted in the release notes, these version numbers are worth looking at:
* `--branch v3.5.13`
  * Specify [the git tag for the release version of Joplin](https://github.com/laurent22/joplin/releases) used to build the webapp
* `FROM node:18`
  * This should match [`/packages/app-mobile/.node-version`](https://github.com/laurent22/joplin/blob/dev/packages/app-mobile/.node-version) for whatever branch is being built
* `FROM nginx:1.29.8-alpine`
  * Update to the most recent stable version of nginx [on Docker Hub](https://hub.docker.com/_/nginx)

If the webapp is enabled in the compose stack, it will automatically rebuild if the Dockerfile changes. A build can be triggered before running as well:
`docker compose -f /path/to/docker-compose.yml build`

### Backups/scheduled exports

The database and filesystem should be backed up using your standard tooling for any container or VM. However, it is also helpful to have a secondary type of export just in case.

To facilitate this, a cron job can be enabled in the compose stack which exports directly to `.jex`. There is also [a plugin available](https://joplinapp.org/plugins/plugin/io.github.jackgruber.backup) which performs a similar task. JEX format exports have several limitations, such as a lack of history, but they could be useful if something goes wrong with your primary backup.

Create the shared export directory and ensure file permissions are set to allow the joplin service user to write there:
```sh
sudo mkdir -p /srv/joplin/data/export/backups
sudo mkdir -p /srv/joplin/data/export/profiles/<username>
sudo chown -R $(id -u):$(id -g) /srv/joplin
sudo chown -R 1001:1001 /srv/joplin/data/export
```

Create a `settings.json` file for each user profile to export based on `backup-profile-settings.json`:
e.g. `/srv/joplin/data/export/profiles/user1@example.com/settings.json`
```json
{
  "sync.target": 9,
  "sync.9.path": "https://subdomain.domain.tld/joplin",
  "sync.9.username": "user1@example.com",
  "sync.9.password": "user_password"
}
```

Notes:
* Sync target `9` means [self-hosted server](https://github.com/laurent22/joplin/blob/1ff71a64e18e1ae3f858281537e10ce678f0ee3d/packages/lib/SyncTargetJoplinServer.ts#L49)
* Username is the user's email address in the admin panel
* This will only work for your personal user account(s) whose passwords you are comfortable saving on the filesystem

Once at least one profile settings file is present, and the compose stack has the `joplin-cron` service & the `export` volume mount enabled, scheduled exports can be saved to the `/srv/joplin/data/export/backups` folder. Exports older than 30 days will be deleted by default. I recommend using a separate backup system such as Syncovery or an rclone script to manage offsite mirroring and monitoring of these exports.

As newer versions of the cli come out (e.g. `npm show joplin versions`), update `joplin@3.5.1` in the backup command defined in cron.

## Deployment

Example deployment will be in `/srv/joplin`, with subdirectories for container configuration (`compose`) and persistent volumes (`data`).

Clone the repository as the `compose` subdirectory:
```sh
sudo mkdir -p /srv/joplin
sudo chown -R $(id -u):$(id -g) /srv/joplin
git clone https://github.com/xenago/joplin.git /srv/joplin/compose
```

Create deployment and data directories owned by the non-root user (any files accessed by the Joplin server must be owned by uid `1001`):
```sh
mkdir -p /srv/joplin/data/joplin/joplin-storage
sudo chown -R 1001:1001 /srv/joplin/data/joplin
```

Prepare the compose stack:
```sh
nano /srv/joplin/compose/docker-compose.yml
```

Note: when a new version of a container is released, update the relevant tag in `docker-compose.yml`

Prepare the environment variables from the example `joplin-server-env`:
```sh
cp /srv/joplin/compose/joplin-server-env /srv/joplin/data/joplin-server-env
nano /srv/joplin/data/joplin-server-env
```

Start it up:
```sh
docker compose -f /srv/joplin/compose/docker-compose.yml up -d
```

Watch the logs if needed:
```sh
docker compose -f /srv/joplin/compose/docker-compose.yml logs -f
```

### Nginx proxy

Create the proxy configuration on your nginx server based on the `joplin-nginx.conf` template. The template configures the webapp interface (mobile app compiled for web) to be served on the base/root path e.g. `https://subdomain.domain.tld` and the server (sync endpoint) to be served in the `/joplin` path, e.g. `https://subdomain.domain.tld/joplin`

Edit the file before reloading the nginx configuration to apply:
- Replace instances of `internal-joplin-host` with actual internal hostname (or IP address) of joplin server
- Replace instances of `subdomain.domain.tld` with actual domain/subdomain
- Omit the base/root server path if not hosting the webapp
- Replace `ssl_certificate` lines entirely, if you are using other certificate paths, such as a wildcard of the base domain

By default, the configuration will limit access to internal IPv4 network IP ranges only. DO NOT MODIFY this behaviour until the server is fully configured in the subsequent step, unless you are using IPv6-only networking (in which case you would need to customize it according to your needs).

### Server configuration

Once deployed, proceed with basic configuration to secure the server.

1. Log into the *server* backend via the proxy, e.g. `https://subdomain.domain.tld/joplin` (not the webapp at the base domain!) - this must be done on the internal network
2. The default admin user has email `admin@localhost` and password `admin`, so change those immediately and save in password manager
3. Create a separate user for personal note sync, don't use the default admin (email can be fake unless you have enabled SMTP, out of scope for this setup).
4. Now that the default credentials have changed, the proxy configuration can optionally be edited to comment out the lines which block external/public access. However, it is generally considered less secure than accessing over VPN.

## Usage

***IMPORTANT: The app-level settings are NOT synced, just the notes!***

Whenever using a new app install, be sure to configure key settings to avoid unintentional loss of history upon sync:
1. The retention period in `Configuration > Note History > Keep note history for` if you do not want it reset to 90 days and wiping out older history
2. The trash setttings in `Configuration > General` - both `Automatically delete notes` and `Keep notes in the trash for` should be configured appropriately if the defaults are not being used

### Webapp

The web frontend can be accessed at `https://subdomain.domain.tld` directly. It doesn't seem to work that well in Firefox, sometimes it thinks there's a second client running in the same browser until refreshed.

### Syncing

Options in `Configuration > Synchronisation`:

* `Synchronisation target`: `Joplin Server`
* `Joplin Server URL`: `https://subdomain.domain.tld/joplin` (the `/joplin` path is important, otherwise the frontend will call itself instead of the server)
* `Joplin Server email`: `<email address previously set for non-admin user`

### Plugins

Joplin clients can load plugins to adjust functionality.

* The available plugins [can be browsed online to some degree](https://joplinapp.org/plugins)
* Not all plugins are compatible with all clients
* Be sure to check the plugin's About page to ensure it has been updated, and validate the source code to ensure the plugin does not do anything too crazy
  * Generally speaking, make sure the plugin doesn't do anything that would break notes with the plugin removed

I use these plugins:
* [Quick Links](https://joplinapp.org/plugins/plugin/com.whatever.quick-links/)
  * Use `@@` to quickly add a link to another other note
* [Easy Backlinks](https://joplinapp.org/plugins/plugin/com.tuibyte.EasyBacklinks/)
  * View notes linking to this one

### Quirks

* Search by Notebook title or tag
  * The normal `F6` search shortcut does not search titles of Notebooks by default
  * Instead, use `Ctrl + P` in combination with `@` or `#` as the first character
  * `@` Will search Notebook Titles
  * `#` Will search tags

### Importing

Marph's [Jimmy](https://github.com/marph91/jimmy) tool can be used to convert from formats other than `.md` for import to Joplin. I tested several methods and this was the easiest by far, though it won't always produce the exact equivalent output and will lack some metadata depending on your original source.

1. Export each note from original source application into its own separate file. I used the `.docx` export function from Samsung Notes to preserve embedded images (it also preserves last-modified time in the file name, but this is not parsed by Jimmy).

2. Place all source files in an input directory with nothing else.

3. Create an empty output directory.

4. Run Jimmy and convert files from the input to the output directory.

5. In the Joplin Desktop application, use the `File > Import > MD - Markdown (Directory)` option, pointing to the output directory used in the previous step.

## License

License only applies to my content; any other content or files mirrored within have their own licenses.
