# Home Assistant Custom Component for Honeywell Evotouch

_News: I am now working towards getting this component accepted into HA; this is the `custom_component` version, which I plan to keep up-to-date._

Support for Honeywell (EU-only) Evohome installations: one controller, multiple heating zones and (optionally) a DHW controller.  It provides _much_ more functionality that the existing Honeywell climate component 

You _could_ run it alongside the existing `honeywell.py` component (but why would you want to?), but you have to edit `honeywell.py`.

Includes support for those of you with multiple locations.  Use can choose _which_ location with `location_idx:`, and you can even have multiple concurrent locations with the following work-around: https://github.com/zxdavb/evohome/issues/10

NB: this is _for EU-based systems only_, it will not work with US-based systems (it can only use the EU-based API).

-DAVB

## Installation instructions (have recently changed)

Make the following changes to your exisiting installation of HA:
 1. Change the `REQUIREMENTS` in /components/honeywell.py to be: `'evohomeclient==0.2.7'` (instead of `0.2.5`) - this will not affect the functionality of that component (only required if you are using this component).
 2. Download this git into the `custom_components` folder (which is under the folder containing `configuration.yaml`) by executing something like: `git clone https://github.com/zxdavb/evohome.git custom_components`
 3. When required, update the git by executing: `git pull` (this git can get very frequent updates).
 4. Edit `configuration.yaml` as below.  I recommend trying 180 seconds, and `high_precision: true` (is default), but YMMV with heuristics/schedules.
 
You will need to redo 1) only after upgrading HA to a later/earlier version.  You will need to do 2) only once.  You will need to redo 3) as often as the git is updated. You will need to do 4) only once.

## Configration file

The `configuration.yaml` is as below (note `evohome:` rather than `climate:` & `- platform: honeywell`).  
```
evohome:
  username: !secret evohome_username
  password: !secret evohome_password

# These config parameters are presented with their default values...
# scan_interval: 300     # seconds, you can probably get away with 60
# high_precision: true   # tenths (hundredths) instead of halves
# location_idx: 0        # if you have more than 1 location, use this

# These config parameters are YMMV...
# use_heuristics: false  # this is for the highly adventurous person, YMMV
# use_schedules: false   # this is for the slightly adventurous person
# away_temp: 15.0        # °C, if you have a non-default Away temp
# off_temp: 5.0          # °C, if you have a non-default Heating Off temp
```
If required, you can add logging as below (make sure you don't end up with two `logger:` directives).
```
# These are for debug logging...
logger:
  logs:
    custom_components.evohome: debug
    custom_components.climate.evohome: debug
#    evohomeclient2: warn
```

## Improvements over the existing Honeywell component

This list is not up-to-date...

1. Uses v2 of the (EU) API via evohome-client: several minor benefits, but v2 temperature precision is reduced from .1C to .5C), so...
2. Leverages v1 of the API to increase precision of reported temps to 0.1C (actually the API reports to 0.01C, but HA only handles 0.1); transparently falls back to v2 temps if unable to get v1 temps. NB: cant do this if >1 location.
3. Exposes the controller as a separate entity (from the zones), and...
4. Correctly assigns operating modes to the controller (e.g. Eco/Away modes) and it's zones (e.g. TemporaryOverride/PermanentOverride modes) - zones state is set to controllers operating mode if in FollowSchedule mode
5. Greater efficiency: loads all entities in a single `add_devices()` call, and uses many fewer api calls to Honeywell during initialisation/polling.
6. The DHW is exposed.
7. Much better reporting of problems communicating with Honeywell's web servers via the client library - entities will report themselves a unavailable in usch scenarios.

## Problems with current implemenation

0. It takes about 60-180 seconds for the client api to accurately report changes made elsewhere in the location (usu. by the Controller).  This is a sad fact if Internet Polling & nothing can be done about it.
1. The controller, which doesn't have a `current_temperature` is implemented as a climate entity, and HA expects all climate entities to report a temperature.  This causes problems with HA.
2. Away mode (as understood by HA), is not implemented as yet.  Away mode is available via the controller.
6. No provision for changing schedules (yet).  This is for a future release.
7. The `scan_interval` parameter defaults to 180 secs, but could be as low as 60 secs.  This is OK as this code polls Honeywell servers only 1x (or 3x) per scan interval (is +2 polls for v1 temperatures), or 60 per hour (plus a few more once hourly).  This compares to the existing evohome implementation, which is at least one poll per zone per scan interval.  I understand that up to 250 polls per hour is considered OK, YMMV.
8. DHW is WIP.  Presently, there is no 'boiler' entity type in HA.
