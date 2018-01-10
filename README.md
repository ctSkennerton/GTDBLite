It uses your username on the server to login so you need to make sure everyone uses their server username in the GTDB.

To set it up you need to do the following (i think - has been a while since I had to do a full setup):

1. Install postgres somewhere. 
2. Logon to the postgres DB as the root or postgres user.
3. Create a database called "gtdb" in the postgres DB.
```
CREATE DATABASE gtdb;
```
3. Create a user called "gtdb" in the postgres DB.
```
CREATE USER gtdb WITH PASSWORD 'somepassword';
```
4. Grant privileges to gtdb user on gtdb db.
```
GRANT ALL PRIVILEGES ON DATABASE gtdb TO gtdb;
```
5. Quit postgres client.

6. Add the base schema. File is in `setup/new_postgres_db.sql` change `INSERT INTO users (username, role_id, has_root_login) VALUES ('user1', 0, true);` to your username on the servers.)

``` 
cat new_postgres_db.sql | psql --username gtdb -h localhost -W -d gtdb
```

7.  If you are going to access from a different server, or multiple servers, you need to update the pg_hba.conf file. Mine is under /etc/postgresql/9.1/main/
```
# Add the following line to the bottom of pg_hba.conf, changing 10.168.48.0/24 to IP address you need to access the DB from.
host     gtdb            gtdb          10.168.48.0/24          md5
```
8.  Create a folder to keep the user genomes and markers. (e.g /db/gtdblite/user_data)
```    
mkdir /db/gtdblite/user_data
mkdir /db/gtdblite/user_data/genomes
mkdir /db/gtdblite/user_data/markers
```
9.  Edit the config file.
```    
GTDB_HOST="<IP address of FQDN of psql server>" # e.g. localhost (if you plan to use it locally)
GTDB_USERNAME="gtdb"
GTDB_DB_NAME="gtdb"
GTDB_PASSWORD="somepassword"
GTDB_GENOME_COPY_DIR="db/gtdblite/user_data/genomes"
GTDB_MARKER_COPY_DIR="db/gtdblite/user_data/markers"
```

10. See if you can connect to the database:
```
gtdblite.py genomes view --all
```
Should say "No genomes found" or something like that.

11. Add in reference genomes. You can download reference genomes from
    NCBI or IMG repositories, it is probably easier to do this from
    NCBI as there is an ftp server of complete bacterial and archaeal
    genomes as the base. All genomes need to be run through CheckM before
    they can be added into the database; save the CheckM results using
    output format 2 (`-o 2`) and with the simple table (`--tab_table`)
    to file. You must also have a four column batchfile that describes
    the genomes that you want to add, the first column is the *full
    path* to the file containing the genome, the second column is the
    name of the genome, the third column is the source of the genome
    (use either "NCBI" or "IMG") and the fourth column is the accession
    of that genome. Note that the fourth column must match the first
    column of the CheckM output. With these two file you can then fun
    the following command:
```
gtdblite.py -r genomes add --batchfile root_user/IMG_genomes.batchfile --checkm_results root_user/checkm.profiles.tsv --create_list "IMG Genomes"
```
    An important thing to note is that when adding in genomes as the root
    user (see the `-r` in the command) the files themselves will not be
    copied into the locations set for user data. Instead only a reference
    to the files is stored in the database. The motovation for this is
    that these genomes probably represent a large amout of disk space that
    we don't want to copy and it is likely that you may want to use these
    files for other purposes than the genome tree database. In contrast,
    when adding in genomes as a user (without the `-r` option above) the
    files will be copied into the genome tree database user data folder.

12. Build your tree
```
        gtdblite.py -t 20 trees create --genome_ids U_67,U_68,U_69 --marker_set_ids 8 --output my_genomes_tree_data --profile_args guaranteed_genome_list_ids=7 --profile_args guaranteed_genome_ids=U_67,U_68,U_69
```
