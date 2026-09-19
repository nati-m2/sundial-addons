# Sundial

Shabbat and holiday automation driven by templates: rules like "the hotplate comes
on two hours before the first meal" are stored once, and the actual times are
recalculated from Home Assistant's own sunset and candle-lighting entities every
time a schedule is built. Nothing has to be edited when the clocks change or the
seasons move.

## Installing

1. Install the **MariaDB** add-on and start it. That is the whole database setup -
   the connection details reach Sundial through the Supervisor, and the schema is
   created on first start.
2. Install Sundial and start it.
3. Open it from the sidebar.

To use a database you already run instead, turn off `use_mariadb_addon` and fill
in the `db_*` options.

## Signing in

There is nothing to sign in to. Ingress has already authenticated whoever opened
the sidebar, and only Home Assistant's own users get that far - so the add-on asks
for no account, no password and no API key, and the Access tab says as much where
a standalone installation would list them.

Running the same image outside the Supervisor is a different matter: there it asks
you to create an administrator account on first run, and both accounts and API
keys are managed on the Access tab. See the project README.

## First run

On the **Settings** screen, choose the entity that holds the current event type
(the `input_select` or similar that says whether today is a regular Shabbat, a
festival, or a guest weekend). The Home Assistant URL and token are not asked for:
the Supervisor supplies both, and they are never stored.

Then, on **Variables**, name the entities whose state is a time - candle lighting,
Shabbat ending, the hour of the first meal. Templates refer to those names, which
is what lets one rule serve every week of the year.

## Knowing that it is Shabbat

Two sources, and the choice is yours. Both stay configured, so switching between
them costs nothing and switching back costs nothing either.

**Home Assistant entities** (the default). A binary sensor - usually
`binary_sensor.jewish_calendar_issur_melacha_in_effect` - decides whether the
scheduler sends commands or only records what it would have sent. Right for a
house that already runs the Jewish Calendar integration.

**The built-in calendar.** Sundial computes the Hebrew date itself and the sunset
at your coordinates, and decides from those. It needs nothing running and nothing
reachable, which matters because candle lighting is exactly when nobody is
available to notice an integration that failed to load or an entity that was
renamed.

Choosing it needs three things you are the authority on, not the software:

| Setting | Why it is yours |
| --- | --- |
| Location | Sunset is computed from it. The **Fill from Home Assistant** button copies the coordinates Home Assistant already has, which is safer than typing them. |
| Candle lighting | Forty minutes before sunset in Jerusalem, thirty in Haifa, eighteen in most other places. |
| End of Shabbat | Either a fixed number of minutes after sunset, or a depression angle - 8.5° is common, and the two differ by more than half an hour. |
| Israel or abroad | Whether a second day of every festival is kept. |

The dashboard shows what the calendar computes **whichever source is in charge**,
along with the Hebrew date and the week's Torah reading. That is deliberate: watch
the two agree for a few weeks before handing the calculation the house. If they
ever disagree while the sensor is deciding, the card says so.

## Options

| Option | Meaning |
| --- | --- |
| `log_level` | `info` is right for normal running. `debug` logs every service call the scheduler makes. |
| `use_mariadb_addon` | Take the database connection from the MariaDB add-on. Leave this on unless you have your own server. |
| `db_host`, `db_port`, `db_user`, `db_password`, `db_name` | An external MySQL or MariaDB. Setting `db_host` overrides the discovery above. |

## What happens when something breaks

The design assumption is that nobody can intervene on Shabbat, so every failure
mode has to degrade rather than stop:

- **The database goes away.** The schedule keeps running. The active plan is held
  in memory and mirrored to a file under `/data`, and the scheduler reads that, not
  the database. What is lost is bookkeeping - the record of what was sent - and it
  catches up when the database returns. The interface says the database is
  unreachable instead of showing an error.
- **Home Assistant restarts.** Commands are retried with a growing delay - 15
  seconds, then 30, then a minute, up to five - so a restart costs a few spaced
  retries rather than exhausting the budget in the twenty seconds it takes the
  scheduler to tick a few times.
- **The add-on restarts, or the machine loses power.** On start, the plan is
  reloaded from the snapshot and any command whose time passed within the catch-up
  window is run. Then every device in the plan is compared against the state it
  should be in and any mismatch is corrected - which is also what repairs a light
  somebody switched off by hand.
- **Something is misconfigured.** A template whose times cannot be resolved is
  dropped with a warning rather than contributing half a pair - an orphan "on"
  leaves a device running indefinitely, and an orphan "off" can switch off
  something a person turned on deliberately.

## Checking a schedule before Shabbat

Build in **simulation** mode. The plan is calculated and displayed exactly as it
would be, and the scheduler walks it and records what it would have done, without
calling Home Assistant. The preview screen shows every event with its computed
time and the template it came from.
