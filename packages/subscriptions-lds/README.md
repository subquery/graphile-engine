# @graphile/subscriptions-lds

PostGraphile schema plugin to provide live updates powered by PostgreSQL
logical decoding. Used as part of PostGraphile's live queries support.

For more background, see [postgraphile issue #92](https://github.com/graphile/postgraphile/issues/92#issuecomment-313476989).

<!-- SPONSORS_BEGIN -->

## Crowd-funded open-source software

To help us develop this software sustainably under the MIT license, we ask
all individuals and businesses that use it to help support its ongoing
maintenance and development via sponsorship.

### [Click here to find out more about sponsors and sponsorship.](https://www.graphile.org/sponsor/)

And please give some love to our featured sponsors 🤩:

<table><tr>
<td align="center"><a href="https://surge.io/"><img src="https://graphile.org/images/sponsors/surge.png" width="90" height="90" alt="Surge" /><br />Surge</a> *</td>
<td align="center"><a href="https://www.netflix.com/"><img src="https://graphile.org/images/sponsors/Netflix.png" width="90" height="90" alt="Netflix" /><br />Netflix</a> *</td>
<td align="center"><a href="https://www.the-guild.dev/"><img src="https://graphile.org/images/sponsors/theguild.png" width="90" height="90" alt="The Guild" /><br />The Guild</a> *</td>
<td align="center"><a href="https://qwick.com/"><img src="https://graphile.org/images/sponsors/qwick.png" width="90" height="90" alt="Qwick" /><br />Qwick</a> *</td>
</tr><tr>
<td align="center"><a href="http://chads.website"><img src="https://graphile.org/images/sponsors/chadf.png" width="90" height="90" alt="Chad Furman" /><br />Chad Furman</a> *</td>
<td align="center"><a href="https://dovetailapp.com/"><img src="https://graphile.org/images/sponsors/dovetail.png" width="90" height="90" alt="Dovetail" /><br />Dovetail</a> *</td>
<td align="center"><a href="https://www.enzuzo.com/"><img src="https://graphile.org/images/sponsors/enzuzo.png" width="90" height="90" alt="Enzuzo" /><br />Enzuzo</a> *</td>
<td align="center"><a href="https://stellate.co/"><img src="https://graphile.org/images/sponsors/Stellate.png" width="90" height="90" alt="Stellate" /><br />Stellate</a> *</td>
</tr></table>

<em>\* Sponsors the entire Graphile suite</em>

<!-- SPONSORS_END -->

## Installation:

Install alongside `postgraphile`, e.g.:

```
yarn add @graphile/subscriptions-lds
```

### Usage:

To use this plugin:

- Ensure your PostgreSQL server has `wal_level = logical` and the `wal2json` plugin (see "Setting up PostgreSQL" below)
- Load this plugin with `--append-plugins` (or `appendPlugins`)
- Enable PostGraphile live with `--live` (or `live: true`)
- If you don't use a superuser or database owner PostgreSQL user with PostGraphile normally (or if you pass a pool to the PostGraphile library), you must provide a superuser or database owner connection string via `--owner-connection` (or `ownerConnectionString`)

CLI:

```
postgraphile \
  --live \
  --owner-connection postgres://db_owner:db_owner_pass@host/db \
  --append-plugins @graphile/subscriptions-lds \
  ...
```

Library:

```js
app.use(
  postgraphile(DB, SCHEMA, {
    // ...

    // Enable live support in PostGraphile
    live: true,
    // We need elevated privileges for logical decoding
    ownerConnectionString: "postgres://db_owner:db_owner_pass@host/db",
    // Add this plugin
    appendPlugins: [
      //...
      require("@graphile/subscriptions-lds").default,
    ],
  })
);
```

## Setting up PostgreSQL

We currently only support PG10+; if you need support for 9.x please get in
touch.

TL;DR: set `wal_level = logical` in `postgresql.conf` and ensure `wal2json`
is installed.

This plugin uses logical decoding and `wal2json`, so you must configure your
PostgreSQL database to support this.

### Setting wal_level = logical

In your `postgresql.conf` you need to enable `wal_level = logical`. You
should ensure that the following settings are set (the `10`s can be any
number greater than 1; set them to how many PostGraphile instances you're
expecting to run, plus a little buffer for regular replication needs):

```
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```

### Installing wal2json

You also need to ensure that `wal2json` is installed. This comes as standard
in many managed PostgreSQL services, such as Amazon RDS, but to install it locally:

1. Ensure that `which pg_config` returns the path to the **correct**
   `pg_config` binary - the one related to your PostgreSQL install. (For
   example, if on a Mac you've installed both Postgres.app _and_ PostgreSQL from
   homebrew then you must modify your `PATH` variable to point at whichever one
   you use, e.g. `export PATH="/Applications/Postgres.app/Contents/Versions/10/bin/:$PATH"`)
2. Run the below code (it takes about 10 seconds):

```bash
git clone https://github.com/eulerto/wal2json.git
cd wal2json
USE_PGXS=1 make
USE_PGXS=1 make install
```

3. (optional) delete the wal2json folder

## Optimising

Please note that the defaults shown below are likely to change over time
based on user feedback. This document will not necessarily be updated with
the new defaults.

### `LD_WAIT` (default 125)

This environmental variable controls how often in milliseconds we check for
changes from the database. Setting it smaller leads to more timely updates
but increases overhead. Setting it larger increases efficiency but means each
batch takes longer to process which may slow the Node.js event loop.

### `LIVE_THROTTLE` (default 500)

This environmental variable is the minimum duration in milliseconds between
live updates to the same subscription.

If your server is getting overwhelmed, you may increase this to increase the
period between live updates sent to clients.

If your application is not responsive enough, you may decrease this to get
closer to real-time updates.

(Throttle fires on both the leading and trailing edge, so decreasing this
only affects successive updates, not the initial update.)

### `LD_TABLE_PATTERN` (default "\*.\*")

Set this envvar to e.g. `app_public.*` to only monitor tables in the
`app_public` schema. See [`filter-tables` in the wal2json
documentation](https://github.com/eulerto/wal2json#parameters)

## Running LDS externally

If you reach sufficient scale that running `@graphile/lds` on its own server
makes sense (rather than using the embedded version) then you can do so
easily. Follow the steps in the `@graphile/lds` README to get the server up
and running, and then set environmental variable `LDS_SERVER_URL` to the full
websocket URL to your LDS server, e.g. `ws://127.0.0.1:9876` (default) before
loading this plugin.

## Compatibility check

You can determine if your PostgreSQL instance is configured correctly with this:

```sql
DO $$
BEGIN
  if current_setting('wal_level') is distinct from 'logical' then
    raise exception 'wal_level must be set to ''logical'', your database has it set to ''%''. Please edit your `%` file and restart PostgreSQL.', current_setting('wal_level'), current_setting('config_file');
  end if;
  if (current_setting('max_replication_slots')::int >= 1) is not true then
    raise exception 'Your max_replication_slots setting is too low, it must be greater than 1. Please edit your `%` file and restart PostgreSQL.', current_setting('config_file');
  end if;
  if (current_setting('max_wal_senders')::int >= 1) is not true then
    raise exception 'Your max_wal_senders setting is too low, it must be greater than 1. Please edit your `%` file and restart PostgreSQL.', current_setting('config_file');
  end if;
  perform pg_create_logical_replication_slot('compatibility_test', 'wal2json');
  perform pg_drop_replication_slot('compatibility_test');
  raise notice 'Everything seems to be in order.';
end;
$$ LANGUAGE plpgsql;
```

If you see the following message then all should be good:

```
NOTICE:  00000: Everything seems to be in order.
```
