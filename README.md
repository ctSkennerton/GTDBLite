# Genome Tree Database
This is a database effort to make it easy to make genome trees 
of various subsets of genomes with various markers. This fork
is behind the times and a [completely revamped version](https://github.com/Ecogenomics/GTDBNCBI)
is under active development elsewhere. Anyone looking to setup
their own database from scratch should look there.

## Installation
It uses your username on the server to login so you need to make sure
everyone uses their server username in the GTDB.  To set it up you need
to do the following (i think - has been a while since I had to do a
full setup):

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
9.  Make the config file. Make the file `src/gtdblite/Config.py` in where ever you downloaded the source code to
    with the following lines. Changing the values to whatever is appropriate for your setup.
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

## Usage

The GTDB is a flexible interface to build genome trees from any
set of stored genomes using any set of stored marker genes. The 
way it accomplishes this is through *lists* of genomes or markers
added in by the user. When building a tree the user can select
one or more *genome lists* and *marker lists* markers will be 
automatically calculated for a genome and stored (so that the
calculation occurs only once).

### Where is data stored?
When adding in a genome or marker to the GTDB the physical files
will be copied to whichever directories you specified in the 
config file for `GTDB_GENOME_COPY_DIR` and `GTDB_MARKER_COPY_DIR`.
This means that if you accidentally delete your genome files there
will still be a copy that you can extract from the GTDB.

The exception to this is the *root user*. Any genomes added by the
root user will not be copied. This is useful for reference genomes
downloaded from NCBI or IMG which will already be quite large and
therefore probably not worth copying.


### Adding genomes

All genomes need to be run through CheckM before
they can be added into the database; save the CheckM results to a file
using the `--tab_table` option. You must also have a three column batchfile that describes
the genomes that you want to add, the first column is the *full
path* to the file containing the genome, the second column is the
name of the genome, the third column is a short description of the genome.
There are also two optional columns: the source of the genome
(use either "NCBI" or "IMG") and the accession
of that genome from the source (i.e the NCBI or IMG accession). 
With these two files you can then run the following command:

```
gtdblite.py genomes add --batchfile <batchfile> --checkm_results <checkm> --create_list <Genome List Name>
```

### Adding tree backbone

(Note: You only need to do this once to get a backbone phylogeny
and you might not need it at all if you only want to make trees
with your own genomes)

Add in reference genomes. You can download reference genomes from
NCBI or IMG repositories, it is probably easier to do this from
NCBI as there is an ftp server of complete bacterial and archaeal
genomes as the base.

#### Downloading genomes from NCBI

The following shell command will download all of the representative
bacteria and archaea genomes from Refseq. It relies on having the
[Entrez Direct](https://www.ncbi.nlm.nih.gov/books/NBK179288/)
e-utilities installed. 
```
esearch -db assembly -query "(txid2157[Organism:exp] OR txid2[Organism:exp]) AND representative [PROP]" |\
  efetch -format docsum |\
  xtract -pattern DocumentSummary -element \
  AssemblyAccession SpeciesTaxid SpeciesName FtpPath_RefSeq |\
  sed 's/,.*//' |\
  sort -k 3,3 |\
  tee downloaded_genomes.tsv |\
  cut -f 4 |\
  sed -e 's/$/\/*genomic.gbff.gz/' |\
  wget -i /dev/stdin
```

#### Adding reference genomes
As mentioned above, when adding in genomes as the root
user the files themselves will not be
copied into the locations set for user data. Instead only a reference
to the files is stored in the database. The motovation for this is
that these genomes probably represent a large amout of disk space that
we don't want to copy and it is likely that you may want to use these
files for other purposes than the genome tree database. 

To add in genomes as the root user:
```
gtdblite.py -r genomes add --batchfile <batchfile> --checkm_results <checkm> --create_list <Genome List Name>
```

### Looking for available data

#### Genome lists
```
gtdblite.py genome_lists view --everyone
```

#### Marker lists
```
gtdblite.py marker_sets view --everyone
```

### Getting an alignment for tree building
Getting an alignment is very flexible it is possible to specify any
combination of individual genome ids, genome list ids, marker ids, and
marker set ids on the command line. Finding out the right genome list id
and marker set id can be achieved using `gtdblite.py genome_lists view`
and `gtdblite.py marker_sets view` sub-commands.
```
gtdblite.py -t 20 alignment create --genome_list_ids 1,2,3 --marker_set_ids 8,1 --output my_genomes_tree_data
```
