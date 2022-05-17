# traefik-geo-ipwhitelist
A bash script to download GeoLite2 country database, extract the IPs for a given list of countries and format into a ipWhiteList middleware file for Traefik.</br>
</br>
When downloading the GeoLite2 database the script saves datetime to local file.
For subsequent runs the script queries Last-Modified http header for GeoLite2 database and only updates the middle ipWhiteList middleware if the database has been modified since the last download. 

## Dependencies
The script uses [unzip](https://manpages.ubuntu.com/manpages/focal/man1/unzip.1.html) to unzip the downloaded zip.
You can check if it is installed by running the below:
```
unzip -v
```

## How to use
0. [Download the script](#0-download-the-script)
1. [Get a Licence Key](#1-get-a-licence-key) 
2. [Update the local variables](#2-update-the-local-variables)
3. [Run the script](#3-run-the-script)
4. [Add middleware to router](#4-add-middleware-to-router)
5. [Restart Traefik](#5-restart-traefik)

### 0. Download the script

Grab the script by your preferred method.</br>
But give a bit of thought where you are going to save as it will remain there alongside the lastModified file.</br>
It will also download the GeoLite2 csv database to the local dir, before extracting the IPs and deleting, if for some reason 20-30mb drive space is significant.

### 1. Get a Licence Key
The geolocation IP list is downloaded from the free GeoLite2 database, [more info here](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data).</br> To access it you need to make an account and get a licence key.

### 2. Update the local variables
Variables you have to change

| Variable           | What it is                            |
| ------------------ | ------------------------------------- |
| countryCodes       | Array of [ISO alpha-2 codes]( https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2#Officially_assigned_code_elements)         |
| maxMindLicenceKey  | Your maxMind licence key              |
| traefikProviderDir | The directory path where your Traefik instance looks for [provider files](https://doc.traefik.io/traefik/providers/file/) |

Variables you might want to change
| Variable             | What it is                            |
| -------------------- | ------------------------------------- |
| middlewareFilename   | The filename for the yml that defines the ipWhitelist middleware |
| middlewareName  | The name for the middleware to be referenced in traefik config        |
| lastModifiedFilename | The filename for the file storing the datetime of when the IP list was last updated |

### 3. Run the script
After it runs you should have file called middleFilename saved at traefikProviderDir.</br>
It should be of the same format as example geo-ipWhiteList.yml

### 4. Add middleware to router
Add the middleware to the desired router. Exact syntax for this varies depending on your [Traefik configuration](https://doc.traefik.io/traefik/middlewares/overview/)</br>
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

### 5. Restart Traefik
