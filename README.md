# Popsn File Server

This is the Popsn file server that hosts encrypted avatars and attachments for the Popsn
network.  It is effectively a "dumb" store of data as it simply stores, retrieves, and expires but
cannot read the encrypted blobs.

It has one additional feature used by Popsn which is acting as a cache and server of the current
version of Popsn (as posted on GitHub) to allow Popsn clients to check the version without
leaking metadata (by making an onion request through the Sispop service node network).

## Requires

### Python3

A reasonably recent version of Python 3 (3.8 or newer are tested, earlier may or may not work), with
the following modules installed.  (Most of these are available as `apt install python3-NAME` on
Debian/Ubuntu).
- flask
- coloredlogs
- psycopg 3.x (*not* the older psycopg2 currently found in most linux distros)
- psycopg_pool
- requests
- uwsgidecorators
- sispopmq
- pyonionreq

(The last two are Sispop projects that either need to be installed manually, or via the Sispop deb
repository).

### WSGI request handler

The file server uses WSGI for incoming HTTP requests.  See below for one possible way to set this
up.

### PostgreSQL

Everything is stored in PostgreSQL; no local file storage is used at all.

## Getting started

0. Create a user, clone the code as a user, run the code as a user, NOT as root.

1. Install the required Python packages:

    ```bash
       sudo apt install python3 python3-flask python3-coloredlogs python3-requests python3-pip

       pip3 install psycopg psycopg_pool  # Or as above, once these enter Debian/Ubuntu
    ```

    You may optionally also install `psycopg_c` for some performance improvements; this likely
    requires first installing libpq-dev for postgresql headers.

2. Build the required sispop Python modules:

       make

3. Set up a postgresql database for the files to live in.  Note that this can be *large* because
   file content is stored in the database, so ensure it is on a filesystem with lots of available
   storage.

   A quick setup, if you've never used it before, is:
   
   ```bash
   sudo apt install postgresql-server postgresql-client  # Install server and client
   sudo su - postgres  # Change to postgres system user
   createuser YOURUSER  # Replace with your username, *NOT* root
   createdb -O YOURUSER popsnfiles  # Creates an empty database for popsn files, owned by you
   exit  # Exit the postgres shell, return to your user

   # Test that postgresql lets us connect to the database:
   echo "select 'hello'" | psql popsnfiles
   # Should should you "ok / ---- / hello"; if it gives an error then something is wrong.

   # Load the database structure (run this from the popsn-file-server dir):
   psql -f schema.pgsql popsnfiles

   # The 'popsnfiles' database is now ready to go.
   ```

4. Copy `fileserver/config.py.sample` to `fileserver/config.py` and edit as needed.  In particular
   you'll need to edit the `pgsql_connect_opts` variable to specify database connection parameters.

5. Set up the application to run via wsgi.  The setup I use is:

   1. Install `uwsgi-emperor` and `uwsgi-plugin-python3`

   1. Configure it by adding `cap = setgid,setuid` and `emperor-tyrant = true` into
      `/etc/uwsgi-emperor/emperor.ini`
   
   1. Create a file `/etc/uwsgi-emperor/vassals/sfs.ini` with content:

      ```ini
      [uwsgi]
      chdir = /home/YOURUSER/Popsn-file-server
      socket = sfs.wsgi
      chmod-socket = 660
      plugins = python3,logfile
      processes = 4
      manage-script-name = true
      mount = /=fileserver.web:app

      logger = file:logfile=/home/YOURUSER/popsn-file-server/sfs.log
      ```

      You will need to change the `chdir` and `logger` paths to match where you have set up the
      code.
    
6. Run:

   ```bash
   sudo chown YOURUSER:www-data /etc/uwsgi-emperor/vassals/sfs.ini
   ```

   (For nginx, you may need to change `www-data` to `nginx` in the above command).

   Because of the configuration you added in step 5, the ownership of the `sfs.ini` determines the
   user and group the program runs as.  Also note that uwsgi sensibly refuses to run as root, but if
   you are contemplating running this program in the first place then hopefully you knew not to do
   that anyway.

7. Set up nginx or apache2 to serve HTTP or HTTPS requests that are handled by the file server.
   - For nginx you want this snippet added to your `/etc/nginx/sites-enabled/SITENAME` file
     (SITENAME can be `default` if you will only use the web server for the Popsn file server).:

     ```nginx
     location / {
         uwsgi_pass unix:///home/YOURUSER/popsn-file-server/sfs.wsgi;
         include uwsgi_params;
     }
     ```

   - If you prefer to use Apache then you want to use a

     ```apache
     ProxyPass / unix:/home/YOURUSER/popsn-file-server/sfs.wsgi|uwsgi://uwsgi-popsn-file-server/
     ```

     directive in `<VirtualHost>` section serving the site.

8. If you want to use HTTPS then set it up in nginx or apache and put the above directives in the
   location for the HTTPS server.  This will work but is *not* required for Popsn and does not
   enhance the security because requests are always onion encrypted; the extra layer of HTTPS
   encryption adds nothing (and makes requests marginally slower).

9. Restart the web server and UWSGI emperor: `systemctl restart nginx uwsgi-emperor`

10. In the future, if you update the file server code and want to restart it, you can just `touch
    /etc/uwsgi-emperor/vassals/sfs.ini` — uwsgi-emperor watches the files for modifications and
    restarts gracefully upon modifications (or in this case simply touching, which updates the
    file's modification time without changing its content).
