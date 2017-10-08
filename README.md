Open a virtual server
====================

**Requirements**
Install the following on an Ubuntu 16.04 server:
 * [HOT Tasking Manager branch custom-damage-assessment assessment](https://github.com/rodekruis/osm-tasking-manager2/tree/custom-damage-assessment), using the [TM installation instructions](https://github.com/rodekruis/osm-tasking-manager2/blob/master/README.md)
 *  [openstreetmap-website branch custom-damage-assessment]( https://github.com/rodekruis/openstreetmap-website/tree/custom-damage-assessment), using the [installation instructions](https://github.com/rodekruis/openstreetmap-website/blob/custom-damage-assessment/INSTALL.md)
 *  [ID-editor branch custom-damage-assessment]( https://github.com/rodekruis/iD/tree/custom-damage-assessment), using the [iD - installation instructions](https://github.com/rodekruis/iD/blob/custom-damage-assessment/README.md)

Tasking manager
========================
If you need to change your custom id-editor url in the tasking manager, edit the file osmtm/static/js/project.js, in [this way](https://github.com/rodekruis/osm-tasking-manager2/commit/97c13833ba7d0005a383ba2c8ec89b3ff4549f89#diff-5150cf14d0a31a305cc80fd521be3569R459)

You need an openstreetmap account to log into the custom tasking manager.

openstreetmap-website
========================
Best is to create one openstreetmap user specifically for the work in ID-editor. The OSM database is an empty copy of openstreetmap.org and does not contain the global user accounts.

**configure openstreetmap-website**
Login to the id-editor with one OSM user and create an OATH ID for this user by going to the user --> profile --> settings --> OATH settings.

Then register a new application with the following
Name: openstreetmap
application-url: http://openstreetmap.missingmaps.nl (or your url)
Map changes: yes

Copy the key into the config/application.yml at "id_key"

**Rebuilding openstreetmap-website**
Whenever you make changes to the openstreetmap-website or ID-editor, you need to perform these actions:
```
cd /var/www/vhosts/openstreetmap-website
rm -rf vendor/assets/iD && bundle exec vendorer
bundle exec rake assets:precompile RAILS_ENV=production
sudo service apache2 restart
```

## restore dump
sudo -u postgres -i
createdb osm_sxm_20170926_2

pg_restore -a -d osm_sxm_20170926_2 building_centroids.backup


## create db
# update database config
nano config/database.yml


bundle exec rake db:create

sudo -u postgres -i

RAILS_ENV=production bundle exec rake db:migrate

export RAILS_ENV=production
rake secret
 export SECRET_KEY_BASE=


#read shapefile
python ogr2osm.py /var/www/vhosts/openstreetmap-website/imports/building_centroids.shp

./osmconvert64 ogr2osm/building_centroids.osm -o=ogr2osm/building_centroids.pbf

# create authfile
host=localhost
database=osm_2
user=postgres
password=pgsql
dbType=postgresql



osmosis --read-pbf-fast selected_buildings.pbf --log-progress --write-apidb authFile=/var/www/vhosts/openstreetmap-website/authFile validateSchemaVersion=no

# clean up
psql -U postgres -d osm -c "select setval('changesets_id_seq', (select max(id) from changesets))"
psql -U postgres -d osm -c "select setval('current_nodes_id_seq', (select max(node_id) from nodes))"
psql -U postgres -d osm -c "select setval('current_ways_id_seq', (select max(way_id) from ways))"
psql -U postgres -d osm -c "select setval('current_relations_id_seq', (select max(relation_id) from relations))"
psql -U postgres -d osm -c "select setval('users_id_seq', (select max(id) from users))"

##postgres
sudo /etc/init.d/postgresql restart

ID-editor
----------------

**Configure ID-editor**
To change the imagery sources that you want to have available:

```
nano data/update_imagery.js
npm install
npm run all
git commit -a -m "update imagery"
git push
```
