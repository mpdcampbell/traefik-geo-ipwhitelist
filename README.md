# traefik-geo-ipwhitelist

A Docker container that creates and updates a GeoIP [ipwhitelist middleware](https://doc.traefik.io/traefik/middlewares/http/ipwhitelist/) file for Traefik.</br>
Takes [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2#Officially_assigned_code_elements) country codes and [ISO 3166-2](https://en.wikipedia.org/wiki/ISO_3166-2#Current_codes) subdivision codes, so access can be restricted at a country, state, city, county level.</br> Also accepts place names, but [this is not recommended](#iso-3166-codes-vs-place-names).</br>
Uses the Maxmind GeoLite2 database and so requires a free [Maxmind account](https://www.maxmind.com/en/geolite2/signup) to work.</br>
</br>
_____
### Tl;dr: How do I use this?
- Make a free maxMind account to get a license key.  
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
- [ISO 3166 codes vs place names](#iso-3166-codes-vs-place-names)
- [Default cron schedule](#default-cron-schedule)

## How does it work?
A bash script downloads the GeoLite2 Country and City databases, reformats and saves a local copy. Then it searches through the database for country/sublocations passed in as environment variables, extracts the matching IPs and formats into an ipWhiteList middleware file for Traefik. If you have Traefik is configured to use file providers, the middleware can then be added to your Traefik router to restrict access to that service to only IPs from the listed locations.

When downloading the databases the last-modified datetime is queried and saved. A cronjob then reruns the script at regular intervals (configurable), each time the last-modified HTTP header for the remote database is queried. The remote database is only downloaded and middleware updated if the database has been modified since the last download.

## Environment Variables

### Mandatory Variables

| Variable           | What it is                            | Example Value          |
| ------------------ | ------------------------------------- |------------------------|
| MAXMIND_KEY        | Your maxmind licence key              | "stringHere"           |
| COUNTRY_CODES      | List of countries you want to allow IPs. <br> See [location formatting] for more details.| FR New-Zealand |
| SUB_CODES | List of locations smaller than a country that you want to allow IPs from. <br> See [location formatting] for more details.|VN-43 West-Virginia:Dallas |

### Optional Variables

| Variable             | What it is                                                                                | Example Value           |
| ---------------------| ----------------------------------------------------------------------------------------- |-------------------------|
| CRON_EXPRESSION      | Overwrites the default cron schedule of</br>0 6 * * wed,sat                               | 5 1 * * MON-FRI         |
| TZ                   | Sets the timezone inside the container, used by cron</br>Default is UTC                   | EDT                     |
| MIDDLEWARE_FILENAME  | The filename of the middleware file written to the provider dir.                          | berlinOnlyMiddleware.yml|
| MIDDLEWARE_NAME      | The name of the middleware to reference inside docker-compose.                            | middleware-berlinOnly   |
| TRAEFIK_PROVIDER_DIR | The directory inside the countainer that the middleware file is written to.</br>Default value /rules| "/path/foldername"      |
| LASTMODIFIED_DIR     | The directory inside the container that the GeoLite2 databases and date last updated timestamps are saved to. </br>Default value /geoip| "/path/foldername"|


## ISO 3166 codes vs place names
ase insensitive.<br>Either ISO-3166-1 Alpha 2 codes or place names.<br>Seperate elements in list by a space and don't use quotes.<br>If a place name contains spaces (i.e. New Zealand) replace the spaces with a dash.

## Default cron schedule

### 3. Run the script
After it runs you should have file called middleFilename saved at traefikProviderDir.</br>
It should be of the same format as example [geo-ipWhiteList.yml](geo-ipwhitelist.yml)

### 4. Add middleware to router
Exact syntax for this varies depending on your [Traefik configuration](https://doc.traefik.io/traefik/middlewares/overview/)</br>
See below a general example for a Docker config, key part is defining "@file" at end of middleware name to inform Traefik the middleware is defined in a file at the provider directory.</br> 
(For example implementation see [L358](https://github.com/mpdcampbell/selfhosted-services/blob/main/docker-compose-traefik.yml#L358) from my selfhosted-services [docker-compose.yml](https://github.com/mpdcampbell/selfhosted-services/blob/main/docker-compose-traefik.yml))

```yml
containerLabel:
  image: containerImage
  labels:
    - "traefik.enable=true"
    # Apply the middleware named "middlewares-geo-ipwhitelist" to the router named "chosen-rtr"
    - "traefik.http.routers.chosen-rtr.middlewares=middlewares-geo-ipwhitelist@file"
```

## To run on a schedule
The GeoLite2 Country database updates every Tuesday. The below commands will set up a cron job every Wednesday at 8am your local time.
You can run the script more often if you'd like. Your maxmind account has a daily limit of 2,000 database downloads but the HEAD request the script first runs to check last-modified datetime doesn't count towards this.

```
crontab -e
```
Then add the below line
```
0 8 * * wed /path to script/geo-ipwhitelist.sh
```
