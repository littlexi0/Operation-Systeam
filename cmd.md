cd ~/postgresql-9.1.3 

make -j

make install -j

$HOME/pgsql/bin/initdb -D $HOME/pgsql/data --locale=C

$HOME/pgsql/bin/postgres -D $HOME/pgsql/data 

$HOME/pgsql/bin/psql postgres -c 'CREATE DATABASE similarity;'

$HOME/pgsql/bin/psql -d similarity -f ./similarity_data.sql

$HOME/pgsql/bin/psql similarity -c "SELECT ra.address, ap.address, ra.name, ap.phone FROM restaurantaddress ra, addressphone ap WHERE levenshtein_distance(ra.address, ap.address) < 4 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%') ORDER BY 1, 2, 3, 4;" > ../levenshtein.txt

$HOME/pgsql/bin/psql similarity -c "SELECT rp.phone, ap.phone, rp.name, ap.address FROM restaurantphone rp, addressphone ap WHERE jaccard_index(rp.phone, ap.phone) > .6 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%') ORDER BY 1, 2, 3, 4; " > ../jaccard.txt

$HOME/pgsql/bin/psql similarity -c "SELECT ra.name, rp.name, ra.address, ap.address, rp.phone, ap.phone FROM restaurantphone rp, restaurantaddress ra, addressphone ap WHERE jaccard_index(rp.phone, ap.phone) >= .55 AND levenshtein_distance(rp.name, ra.name) <= 5 AND jaccard_index(ra.address, ap.address) >= .6 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%')ORDER BY 1, 2, 3, 4, 5, 6;" > ../combined.txt

$HOME/pgsql/bin/pg_ctl -D $HOME/pgsql/data stop
