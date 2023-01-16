# [<img alt="alt_text" width="50px" src="https://www.codeslikeaduck.com/img/codeDuck.svg" />](https://www.codeslikeaduck.com/)  traefik-geo-ipwhitelist <br> [![License](https://img.shields.io/badge/license-BSD%202--Clause-blue)](https://github.com/mpdcampbell/traefik-geo-ipwhitelist/blob/main/LICENSE) [![Docker Pulls](https://img.shields.io/docker/pulls/mpdcampbell/traefik-geo-ipwhitelist?color=red)](https://hub.docker.com/r/mpdcampbell/traefik-geo-ipwhitelist)

A Docker container that creates and updates a GeoIP [ipwhitelist middleware](https://doc.traefik.io/traefik/middlewares/http/ipwhitelist/) file for Traefik.</br>
Uses the Maxmind GeoLite2 database and so requires a free [MaxMind account](https://www.maxmind.com/en/geolite2/signup) to work.</br>
Access can be restricted at country, state, county, city or town level (with decreasing accuracy).</br>
Accepts [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2#Officially_assigned_code_elements) country codes, [ISO 3166-2](https://en.wikipedia.org/wiki/ISO_3166-2#Current_codes) subdivision codes, and [place names](#formatting-iso-3166-codes-and-place-names).</br>
</br>
_____
### TL;DR: How do I use this?
- Make a free MaxMind account to get a license key.  
- Download [docker-compose.example.yml](/docker-compose.example.yml) and add the lines to your traefik config as instructed.  
- Replace the dummy paths and key in the example.  
- Replace the location variables, countries go in COUNTRY_CODES, locations smaller than countries go in SUB_CODES.  
- Start up the container with ``docker-compose -f docker-compose.example.yml up -d``
- Check the logs with ``docker logs -tf geoipwhitelist`` to confirm its working.
_____
<br>  

## Contents
- [How does it work?](#how-does-it-work)
- [Environment variables](#environment-variables)
- [Formatting ISO 3166 codes and place names](#formatting-iso-3166-codes-and-place-names)
- [Default cron schedule](#default-cron-schedule)
- [License](#license)

## How does it work?
A bash script downloads the GeoLite2 Country and City databases, reformats them and saves a local copy. Then it searches through the database for country/sublocations passed in as environment variables, extracts the matching IPs and formats into an ipWhiteList middleware file for Traefik. This is written down to the provider directory outside the container. With Traefik configured to use file providers, the middleware can then be added to a router to restrict access to that service to only IPs from the listed locations.

When downloading the databases the last-modified datetime is queried and saved. A cron job then reruns the script at regular intervals (configurable) and each time the last-modified HTTP header for the remote database is queried. The remote database is only downloaded and the middleware updated if the database has been modified since the last download.

## Environment Variables

### Mandatory Variables

| Variable           | What it is                            | Example Value          |
| ------------------ | ------------------------------------- |------------------------|
| MAXMIND_KEY        | Your MaxMind license key              | stringHere           |
| COUNTRY_CODES      | List of countries you want to allow IPs from. <br> See [formatting](#country_codes) for more details.| FR New-Zealand |
| SUB_CODES | List of locations smaller than a country that you want to allow IPs from. <br> See [formatting](#sub_codes) for more details.|VN-43 West-Virginia:Dallas |

### Optional Variables

| Variable             | What it is                                                                                | Example Value           |
| ---------------------| ----------------------------------------------------------------------------------------- |-------------------------|
| CRON_EXPRESSION      | Overwrites the default cron schedule of ```0 6 * * wed,sat```                             | 5 1 * * MON-FRI         |
| TZ                   | Sets the timezone inside the container, used by cron.</br>Default is UTC                   | EDT                     |
| MIDDLEWARE_FILENAME  | The filename of the middleware file written to the provider dir.                          | berlinOnlyMiddleware.yml|
| MIDDLEWARE_NAME      | The name of the middleware to reference inside docker-compose.                            | middleware-berlinOnly   |
| TRAEFIK_PROVIDER_DIR | The directory inside the container that the middleware file is written to.</br>Default value /rules| /path/foldername      |
| LASTMODIFIED_DIR     | The directory inside the container that the GeoLite2 databases and date last updated timestamps are saved to by default. </br>Default value /geoip| "/path/foldername"|
| COUNTRY_DIR | The directory inside the container that the country database file is saved to.</br>Default value LASTMODIFIED_DIR/country| /path/foldername      |
| SUB_DIR | The directory inside the container that the subdivision database file is saved to.</br>Default value LASTMODIFIED_DIR/sub| /path/foldername      |
<br>

## Formatting ISO 3166 codes and place names
### COUNTRY_CODES
- Enter the countries you want to allow as either [ISO-3166-1 Alpha 2 codes](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2#Officially_assigned_code_elements) or the place name. Using ISO codes is recommended as they are unambiguous. Place names and their spellings can vary regionally and is more likely to lead to errors.<br>
- Seperate elements in the list with a space.<br>
- If a place name contains spaces (i.e. New Zealand) replace the spaces with a dash (i.e. New-Zealand)<br>
- Don't use quotation marks.<br>
- The list is case insensitive.<br>

### SUB_CODES
**Note: There is no guarantee the sublocation you wish to limit access to is listed in the GeoLite2 database.**<br>
<br>
Accepts [ISO-3166-2 codes](https://en.wikipedia.org/wiki/ISO_3166-2#Current_codes) but the GeoLite database also lists IP address by smaller areas. For example in the United States the ISO-3166-2 codes represent states, when you might want to limit access to a given city or town. For this reasion the variable also accepts place names, however they should always be qualified with the larger region. Take Berlin as an example. 29 locations in the GeoLite2 database have Berlin in their name including towns in Russia, Uruguay, Colombia, and the United States. To narrow this down, the SUB_CODES variable accepts place names in the form ```Larger-Region:Location```.<br>
<br>
For example:<br>
```United-States:Berlin``` - This will match all the listed towns in the United States named Berlin.<br>
```Wisconsin:Berlin``` - This will match the listed towns in Wisconsin named Berlin.<br>
```Wisconsin:New-Berlin``` - This will match the town New Berlin in Wisconsin, which wasn't in the previous example.<br> 
Please note that obviously all towns and regions in the world are not in the database. Also regional spelling can vary. In general using place names is much more hit-or-miss than using ISO codes. You can check what is listed by having a grep in the subList.txt file inside SUB_DIR.<br>
<br>
Also the same format rules as for COUNTRY_CODES apply:
- Seperate elements in the list with a space.<br>
- If a place name contains spaces (i.e. New Berlin) replace the spaces with a dash (i.e. New-Berlin)<br>
- Don't use quotation marks.<br>
- The list is case insensitive.<br>

## Default cron schedule
By default the container adds a cron job to run the script at 6 AM UTC on Wednesdays and Saturdays. This is because the MaxMind Geolite 2 country and city databases update every [Tuesday and Friday.](https://support.maxmind.com/hc/en-us/articles/4408216129947) If you want to change the schedule you can define your own [cron expression](https://crontab.cronhub.io/) in the CRON_EXPRESSION environment variable, which will overwrite the default schedule. The cron job will run with the default timezone, UTC, but you can change this with the TZ environment variable.<br>
<br>
The free MaxMind account has a daily limit of 2,000 database downloads but the script first runs a HEAD request, to check if the last-modified header has changed, which doesn't count towards this limit. The script should only download the database if the last-modified is more recent than the last-modified time for the local database copies.

## License

[BSD 2-Clause License](/LICENSE)
