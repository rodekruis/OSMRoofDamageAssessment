# Roof damage assessment by aerial imagery
--------------------
Using high resolution aerial imagery it is possible to do a crowdsourced quick damage assessment after a disaster that has done visible damage to roofs.

This readme explains how to set up the HOTOSM tasking manager, openstreetmap-website and id-editor to allow a small group of  trained and instructed mappers to classify damage, roof types and roof materials.

The setup described in this readme has three classification tasks:
- Roof type: classify hipped, flat and pitch roofs based on pre-disaster imagery (imagery: 20cm is good, 50cm is not great)
- Roof material: classify materials that exist in the area. Current setup is: concrete, metal, tiles, mixed, tiles or metal (if undecided), and unknown
- Damage: classify damage using the [Harvard Humanitarian Initiative BAR methodology](https://hhi.harvard.edu/publications/satellite-imagery-interpretation-guide-assessing-wind-disaster-damage-structures), in five categories: none, partial, significant, destroyed, unknown. (imagery: <10cm is ideal, 20cm maybe possible, 50cm not good).

# Requirements
--------------------
Install the following on an Ubuntu 16.04 server:
 * [HOT Tasking Manager branch custom-damage-assessment assessment](https://github.com/rodekruis/osm-tasking-manager2/tree/custom-damage-assessment), using the [TM installation instructions](https://github.com/rodekruis/osm-tasking-manager2/blob/master/README.md). Install in /var/www/vhosts/tm/
 *  [openstreetmap-website branch custom-damage-assessment]( https://github.com/rodekruis/openstreetmap-website/tree/custom-damage-assessment), using the [installation instructions](https://github.com/rodekruis/openstreetmap-website/blob/custom-damage-assessment/INSTALL.md). Install in /var/www/vhosts/openstreetmap-website
 *  [ID-editor branch custom-damage-assessment]( https://github.com/rodekruis/iD/tree/custom-damage-assessment), using the [iD - installation instructions](https://github.com/rodekruis/iD/blob/custom-damage-assessment/README.md). Install in /var/www/vhosts/iD
- ogr2osm, using [this installation instruction](http://wiki.openstreetmap.org/wiki/Ogr2osm). Install in /root/
- OSMConvert, download the 64bit linux binary, using [these installation instructions](http://wiki.openstreetmap.org/wiki/Osmconvert). Install in /root/

Install Apache
```
sudo apt-get update
sudo apt-get install apache2

sudo apt-get install libapache2-mod-passenger
sudo a2enmod passenger
```

# Tasking manager
----------------------

If you need to change your custom id-editor url in the tasking manager, edit the file osmtm/static/js/project.js, in [this way](https://github.com/rodekruis/osm-tasking-manager2/commit/97c13833ba7d0005a383ba2c8ec89b3ff4549f89#diff-5150cf14d0a31a305cc80fd521be3569R459)

You need an openstreetmap account to log into the custom tasking manager.

## configure apache for the tasking manager
Add a site to /etc/apache2/sites-available/tm.conf with the following content, and update the servername and serveradmin:
```
WSGIRestrictStdin Off
<VirtualHost *:80>
ServerName "tasks.your.domain"
ServerAdmin info@your.dommain
DocumentRoot /var/www/vhosts/tm/
CustomLog /var/log/apache2/osmtm-access.log combined
ErrorLog /var/log/apache2/osmtm-error.log

WSGIDaemonProcess tm user=www-data group=www-data threads=5 display-name=%{GROUP} python-path=/var/www/vhosts/tm/:/var/www/vhosts/tm/env/lib/python2.7/site-packages
WSGIScriptAlias / /var/www/vhosts/tm/production.wsgi

<Directory /var/www/vhosts/tm/>
        WSGIApplicationGroup %{GLOBAL}
        WSGIProcessGroup tm
        WSGIPassAuthorization On

        Options Indexes FollowSymLinks
        AllowOverride All
        Order deny,allow
        Allow from all
        IndexOptions FancyIndexing
</Directory>
</VirtualHost>
```
Enable the site with:
```
a2ensite tm
sudo service apache2 restart
```


# openstreetmap-website
----------------------

## configure apache for openstreetmap-website
Add a site to /etc/apache2/sites-available/openstreetmap-website.conf with the following content, , and update the servername and serveradmin:
```
<VirtualHost *:80>
      ServerAdmin info@your.domain
      ServerName openstreetmap.your.domain
      DocumentRoot /var/www/vhosts/openstreetmap-website/public
      PassengerRuby /usr/bin/ruby
    <Directory /var/www/vhosts/openstreetmap-website/public>
      Allow from all
      Options -MultiViews
      # Uncomment this if you're on Apache >= 2.4:
      Require all granted
    </Directory>
</VirtualHost>
```
Enable the site with:
```
a2ensite openstreetmap-website
sudo service apache2 restart
```

## configure openstreetmap-website
Best is to create one openstreetmap user specifically for the work in ID-editor. The OSM database is an empty copy of openstreetmap.org and does not contain the global user accounts.

Login to the id-editor with one OSM user and create an OATH ID for this user by going to the user --> profile --> settings --> OATH settings.

Then register a new application with the following:
```
Name: openstreetmap
application-url: http://openstreetmap.missingmaps.nl (or your url)
Map changes: yes
```
Copy the key into the config/application.yml at "id_key"

## Rebuilding openstreetmap-website
Whenever you make changes to the openstreetmap-website or ID-editor, you need to perform these actions:
```
cd /var/www/vhosts/openstreetmap-website
rm -rf vendor/assets/iD && bundle exec vendorer
bundle exec rake assets:precompile RAILS_ENV=production
sudo service apache2 restart
```

## Prepare damage OSM database

### Configure secrets
Edit config/secrets.yml. Perform the below, for each environment.
```
export RAILS_ENV=production
rake secret
## replace secret key at production 
nano config/secrets.yml
 ```
 
### Create database to import nodes into
Update the database config. and set three different database names for development, test and production.
```
nano config/database.yml
```
Create the database
```
bundle exec rake db:create
```
### import building nodes
Roof damage classification works best if an empty OSM database is created with one node on the centroid of each building. This therefor requires a very accurate base map in OSM. Use QGIS to create a shapefile with one point per building. Make sure to include the OSM identified of your building in an osm_id column of your centroid shapefile. This you need later to bind the data to the original OSM building polygons, and enrich OSM with your data.

Upload the shapefile to the server and convert it to a .pbf file:
```
python ogr2osm.py building_centroids.shp
./osmconvert64 ogr2osm/building_centroids.osm -o=ogr2osm/building_centroids.pbf
```

Create an authfile in /root/authFile
```
##nano /root/authFile
host=localhost
database=osm
user=postgres
password=pgsql
dbType=postgresql
```
Run this command to import the building points as nodes in the osm database:
```
osmosis --read-pbf-fast building_centroids.pbf --log-progress --write-apidb authFile=/root/authFile validateSchemaVersion=no
```

## clean up
Run the following commands to set the sequences in the database:
```
psql -U postgres -d osm -c "select setval('changesets_id_seq', (select max(id) from changesets))"
psql -U postgres -d osm -c "select setval('current_nodes_id_seq', (select max(node_id) from nodes))"
psql -U postgres -d osm -c "select setval('current_ways_id_seq', (select max(way_id) from ways))"
psql -U postgres -d osm -c "select setval('current_relations_id_seq', (select max(relation_id) from relations))"
psql -U postgres -d osm -c "select setval('users_id_seq', (select max(id) from users))"
```

## Optimize postgres
Edit /etc/postgresql/postgresql.conf. Look for the below settings and replace the values. {{values}} should be replaced also.
```
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 100
pg_stat_statements.track = top
pg_stat_statements.save = off
shared_buffers = {{shared_buffers}}MB # 25% of system memory
effective_cache_size = {{effective_cache_size}}MB # 50% of system memory
checkpoint_completion_target = 0.9
random_page_cost = 2.0
work_mem = 30MB
maintenance_work_mem = 300MB
max_stack_depth = 3MB
```
## Restart postgres
```
sudo /etc/init.d/postgresql restart
```

## Upgrading openstreetmap-website
To upgrade the custom-damage-assessment branch of the openstreetmap-website to the master of the [official openstreetmap-website repository](https://github.com/openstreetmap/openstreetmap-website/), do the following:
```
cd /var/www/vhosts/openstreetmap-website
git remote add openstreetmap https://github.com/openstreetmap/iD.git #probably already done
git fetch openstreetmap
git merge openstreetmap-website/release
git commit -a -m "merge with master"
git push
```

# ID-editor
----------------------
The ID-editor is built from the /var/www/vhosts/iD working copy, and then pushed to the repository on github. The openstreetmap-website will pull in the build code of the ID-editor, and launch the ID-editor within the openstreetmap-website. No additional configuration of apache for the ID-editor is needed.

### Imagery sources
To change the imagery sources that you want to have available:

```
nano data/update_imagery.js
npm install
npm run all
git commit -a -m "update imagery"
git push
```

### Filters
A good practice is to set up feature filters, so that in the ID-editor you can filter the building nodes based on the tags. In case you need to reclassify damage to specific types of buildings this comes in useful.

```
 cd /var/www/vhosts/iD/
 nano modules/renderer/features.js
 ## add, remove or change feature filters
 ```
 Make sure that for any added filters you also add the translation files. Use [the examples](https://github.com/rodekruis/iD/blob/custom-damage-assessment/data/core.yaml#L425-L469).
 ```
 nano data/core.yaml
 ```
Rebuild id-editor:
```
npm run all
git commit -a -m "updated filters"
git push
```
For the changes to be added to the openstreetmap-website, you need also to rebuild the openstreetmap-website working copy using the steps in #Rebuilding openstreetmap-website

## Upgrading ID-editor
To upgrade the custom-damage-assessment branch of iD to the [current latest release](https://github.com/openstreetmap/iD/releases) of the [official iD repository](https://github.com/openstreetmap/iD/), do the following:
```
cd /var/www/vhosts/iD
git remote add latest-id https://github.com/openstreetmap/iD.git
git fetch latest-id
git merge v2.4.1 # replace with latest release
npm install
npm run all
git commit -a -m "merge with latest release"
git push
```

# Analyzing damage data
---------------------

Roof type, material and damage data can be extracted from the OSM database with the following commands on the server:

``
psql -U postgres -d osm
``

And execute this query:

```sql
Copy (
select no.node_id, no.version,no.latitude, no.longitude, no.tile, nt3.v as osm_id, nt1.v as damage, nt2.v as roof, nt4.v as material
FROM 
nodes no
LEFT JOIN node_tags nt1
ON nt1.node_id = no.node_id AND nt1.version = no.version AND nt1.k='damage'
LEFT JOIN node_tags nt2
ON nt2.node_id = no.node_id AND nt2.version = no.version AND nt2.k='roof'
LEFT JOIN node_tags nt4
ON nt4.node_id = no.node_id AND nt4.version = no.version AND nt4.k='material'
LEFT JOIN node_tags nt3
ON nt3.node_id = no.node_id AND nt3.version = no.version AND nt3.k='osm_id'
WHERE
no.version = (SELECT MAX(version) FROM nodes n WHERE n.node_id = no.node_id GROUP BY node_id)
ORDER BY node_id ASC
) To '/tmp/damagedata.csv' With CSV DELIMITER ',';
```

Grab the file from /tmp and use it in your analysis. It contains for every node the osm_id (refering to the original building polygon in OSM), the damage, roof type and roof material classifications, and the node_id.
