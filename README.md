# Pleroma-Docker (Unofficial)

[Pleroma](https://pleroma.social/) is a selfhosted social network that uses ActivityPub.

This repository dockerizes it for easier deployment.

<hr>

```cpp
#include <LICENSE>

/*
 * This repository comes with ABSOLUTELY NO WARRANTY
 *
 * I will happily help you with issues related to this script,
 * but I am not responsible for burning servers, angry users, fedi drama,
 * thermonuclear war, or you getting fired because your boss saw your NSFW posts.
 *
 * Please do some research if you have any concerns about the
 * included features or software *before* using it.
 */
```

<hr>

## In the Wild

`cofe.rocks` is always managed by this script.<br/>
Take a look at [hosted/pleroma](/hosted/pleroma) if you get stuck or need some inspiration.

Additionally it's known to run on (in no particular order):
- social.interhop.org
- social.technodruide.ca
- is.badat.dev

Does your instance use pleroma-docker?<br/>
Let me know and I'll add you to this list.

## Docs

These docs assume that you have at least a basic understanding
of the pleroma installation process and common docker commands.

If you have questions about Pleroma head over to https://docs.pleroma.social/.
For help with docker check out https://docs.docker.com/.

For other problems related to this script, contact me or open an issue :)

### Prerequisites

- ~1GB of free HDD space
- `git` if you want smart build caches
- `curl`, `jq`, and `dialog` if you want to use `./manage.sh mod`
- Bash 4+
- Docker 18.06+ and docker-compose 1.22+

### Installation

- Clone this repository
- Create a `config.exs` and `.env` file
- Run `./manage.sh build` and `./manage.sh up`
- [Configure a reverse-proxy](#my-instance-is-up-how-do-i-reach-it)
- Profit!

Hint:
You can also use normal `docker-compose` commands to maintain your setup.<br/>
The only command that you cannot use is `docker-compose build` due to build caching.

### Configuration

All the pleroma options that you usually put into your `*.secret.exs` now go into `config.exs`.<br/>
`.env` stores config values that need to be known at orchestration/build time.<br/>
Documentation for the possible values is inside of that file.

### Updates

Run `./manage.sh build` again and start the updated image with `./manage.sh up`.<br/>
You don't need to stop your pleroma server for either of those commands.

### Maintenance

Pleroma maintenance is usually done with mix tasks.<br/>
You can run these tasks in your running pleroma server using `./manage.sh mix [task] [arguments...]`.<br/>
For example: `./manage.sh mix pleroma.user new sn0w ...`<br/>
If you need to fix bigger problems you can also spawn a shell with `./manage.sh enter`.

### Postgres Upgrades

Postgres upgrades are a slow process in docker (even more than usual) because we can't utilize `pg_upgrade` in any sensible way.<br/>
If you ever wish to upgrade postgres to a new major release for some reason, here's a list of things you'll need to do.

- Inform your users about the impending downtime
    - Seriously this can take anywhere from a couple hours to a week depending on your instance
- Make sure you have enough free disk space or some network drive to dump to, we can't do in-place upgrades
- Stop pleroma (`docker-compose stop server`)
- Dump the current database into an SQL file (`docker-compose exec db pg_dumpall -U pleroma > /my/sql/location/pleroma.sql`)
- Remove the old containers (`docker-compose down`)
- Modify the postgres version in `docker-compose.yml` to your desired release
- Delete `data/db` or move it into some different place (might be handy if you want to abort/revert the migration)
- Start the new postgres container (`docker-compose up -d db`)
- Start the import (`docker-compose exec -T db psql -U pleroma < /my/sql/location/pleroma.sql`)
- Wait for a possibly ridculously long time
- Boot pleroma again (`docker-compose up -d`)
- Wait for service to stabilize while federation catches up
- Done!

### Customization

Add your customizations (and their folder structure) to `custom.d/`.<br/>
They will be copied into the right place when the container starts.<br/>
You can even replace/patch pleroma’s code with this, because the project is recompiled at startup if needed.

In general: Prepending `custom.d/` to pleroma’s customization guides should work all the time.<br/>
Check them out in the [pleroma documentation](https://docs.pleroma.social/small_customizations.html#content).

For example: A custom thumbnail now goes into `custom.d/` + `instance/static/instance/thumbnail.jpeg`.

### Patches

Works exactly like customization, but we have a neat little helper here.<br/>
Use `./manage.sh mod [regex]` to mod any file that ships with pleroma, without having to type the complete path.

### My instance is up, how do I reach it?

To reach Gopher or SSH, just uncomment the port-forward in your `docker-compose.yml`.

To reach HTTP you will have to configure a "reverse-proxy".<br/>
Older versions of this project contained a huge amount of scripting to support all kinds of reverse-proxy setups.<br/>
This newer version tries to focus only on providing good pleroma tooling.<br/>
That makes the whole process a bit more manual, but also more flexible.

You can use Caddy, Traefik, Apache, nginx, or whatever else you come up with.<br/>
Just modify your `docker-compose.yml` accordingly.

One example would be to add an [nginx server](https://hub.docker.com/_/nginx) to your `docker-compose.yml`:
```yml
  # ...

  proxy:
    image: nginx
    init: true
    restart: unless-stopped
    links:
      - server
    volumes:
      - ./my-nginx-config.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
      - "443:443"
```

Then take a look at [the pleroma nginx example](https://git.pleroma.social/pleroma/pleroma/blob/develop/installation/pleroma.nginx) for hints about what to put into `my-nginx-config.conf`.

Using apache would work in a very similar way (see [Apache Docker Docs](https://hub.docker.com/_/httpd) and [the pleroma apache example](https://git.pleroma.social/pleroma/pleroma/blob/develop/installation/pleroma-apache.conf)).

The target that you proxy to is called `http://server:4000/`.<br/>
This will work automagically when the proxy also lives inside of docker.

If you need help with this, or if you think that this needs more documentation, please let me know.

Something that cofe.rocks uses is simple port-forwarding of the `server` container to the host's `127.0.0.1`.<br/>
From there on, the natively installed nginx server acts as a proxy to the open internet.<br/>
You can take a look at cofe's [compose yaml](/hosted/pleroma/src/branch/master/docker-compose.yml) and [proxy config](/hosted/pleroma/src/branch/master/proxy.xconf) if that setup sounds interesting.

### Attribution

Thanks to [Angristan](https://github.com/Angristan/dockerfiles/tree/master/pleroma) and [RX14](https://github.com/RX14/kurisu.rx14.co.uk/blob/master/services/iscute.moe/pleroma/Dockerfile) for their dockerfiles, which served as an inspiration for the early versions of this script.

The current version is based on the [offical install instructions](https://docs.pleroma.social/alpine_linux_en.html).
Thanks to all people who contributed to those.