# matomo-docker

This repository allows you to quickly install Matomo, an analytics platform tool, using docker-compose, and configure it to track the behavior of the users of your preferred website or online application.

## Installation instructions

To get started, download and install Docker from www.docker.com.

### Step 0 - Download this repo

Clone the repository:
```
git clone https://github.com/thehyve/matomo-docker.git
```

### Step 1 - Make sure docker-compose is installed

See [https://docs.docker.com/compose/install/#install-compose](https://docs.docker.com/compose/install/#install-compose).

### Step 2 - Change the address of the VIRTUAL_HOST

Go to the `docker-compose.yml` file and change the `VIRTUAL_HOST` value (environment variable of the `matomo-app` container). Now it is set to `https://localhost:8443`, which works when testing locally, but change it to the appropriate host if you want to set up the application in a different place.

### Step 3 - Configure apache reverse HTTPS proxy

Insert the hostname component of the URL (without `http://`) into `docker-compose.yml`, on the line that says `APACHE_PROXY_HOSTNAME=`.

For testing, create self-signed certificates by running this command:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key.key -out cert.crt -subj '/CN=localhost'
```

And place them in the `apache-proxy/` folder.
For production, replace `cert.crt` and `key.key` by real certificates for the public URLs.

### Step 4 - Build and initialize containers

Do an initial build and start of the containers with:
```
cd matomo-docker/
docker-compose build
docker-compose up -d
```

:warning: Three folders called `config`, `logs` and `matomo_db` will be created in your repository instance. If you want to start an installation from scratch later, you will need to remove them.

### Step 5 - Open the Matomo app and follow the installation steps

In your browser, go to the address specified in `VIRTUAL_HOST` (default: `https://localhost:8443/`), and you will see the Matomo Welcome Screen. Follow the installation steps that are shown in the screen:
- For Step 1 and Step 2 you don't need to change anything, just press `Next`.
- In `Step 3: Database Setup`, change the values of the following parameters:

    | Parameter | Value |
    | ----- | ----- |
    | Database Server | matomo-db |
    | Login | matomo |
    | Password | P@ssword1 |
    | Database Name | matomo |
    
  Leave the rest as is and click `Next`.
- In Step 4 you should see a `Tables created with success!` message, you can click `Next` to continue.
- In Step 5 you need to enter the credentials for the Super User,
  that will be the user used to log in into the Matomo instance.
  Enter a nonsense email (`noreply@example.com`),
  as none of the functionality we use actually seems to involve
  sending anything to the address,
  and hit `Next`.
- In Step 6, you are going to set up a Website to track. You can add more websites to track later. Once you have filled the fields, click `Next`.
- In Step 7, you will see the JavaScript Tracking Code for the website you configured in Step 6. This piece of code is used to track what the users are doing in your website and report it to Matomo. Therefore, you must copy the code and paste it inside the `<header>` section of every page of your website. This JavaScript Tracking code can be customized to specifically track what you want or deal with different types of websites, more information can be found [here](https://developer.matomo.org/guides/tracking-javascript-guide). For customizing cBioPortal, check the section [Integration of Matomo with cBioPortal](#integration-of-matomo-with-cbioportal) below. Once you finished installing the JavaScript Tracking Code on your website, click `Next`.
- At the end of the wizard you will see a `Congratulations` page. Click `Continue to Matomo` to access the Matomo's log in page, where you should log in with your Super User credentials. This is the page you will see from now on every time you access Matomo via the address specified in `VIRTUAL_HOST` (default: `https://localhost:8443/`). If you see a warning message instead of the login page, you need to adjust the `config.init.php` file (explained in the following section: step 6).

Once logged in to the Matomo admin panel,
head to the Administration icon in the top-right corner,
and then to System > Geolocation in the left-hand menu.
Enable the second option, GeoIP 2 (PHP).  
This will still only roughly track users' locations,
as the last two bytes of their IP addresses are anonymized
by the default settings under Privacy > Anonymize data.

### Step 6 - Edit the config.init.php file
Go to the folder `config`:
```
cd config
```

Open the `config.init.php` file and replace this line (under the `[General]` section):
```
trusted_hosts[] = "localhost"
```

by this:
```
trusted_hosts[] = "localhost:8443"
```

### All set
Once you log in, you will access to Matomo's Dashboard, where you can see the tracked data for your websites, add more websites to track, change the configuration, etc.

## Integration of Matomo with cBioPortal

To use Matomo for tracking cBioPortal, you need to adjust a few parameters of the JavaScript Tracking Code. First of all, the JavaScript Tracking Code must be added in the `<header>` section of the file `portal/src/main/webapp/index.jsp`.

Since cBioPortal is a single page application, to make Matomo track the visits to the different "pages" of the application (results, study view, patient view), we need to add this code inside the JavaScript Tracking Code:

```
// store url on load
var currentPage = window.location.href;

// listen for changes
setInterval(function()
{
    if (currentPage != window.location.href)
    {
        // page has changed, set new page as 'current'
        currentPage = window.location.href;

        // register as new view
        _paq.push(['setCustomUrl', window.location.href]);
        _paq.push(['setDocumentTitle', window.document.title]);
        _paq.push(['setGenerationTimeMs', 0]);
        _paq.push(['trackPageView']);
    }
}, 500);
```

This code checks every 500 ms if the URL of the page has changed, and if so, it adds the new URL as a new "page" visited.

Finally, if you are using cBioPortal with Keycloak (or another authentication application) and you want to know the identity of the users, add this line below the `setSiteId`:

```
_paq.push(['setUserId', "<%=GlobalProperties.getAuthenticatedUserName()%>"]);
```

This code sets the username of the authentication application as the User ID of Matomo.
