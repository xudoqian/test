Certainly! Here's an updated version of the script that displays usage instructions when the script is run with incorrect parameters or when the `--help` or `-h` flag is provided:

```bash
#!/bin/bash

# Function to display usage instructions
display_usage() {
    echo "Usage: $0 USERNAME PASSWORD REPLICA_SET_URI PORT FOLDER_PATH DATABASE_LIST_FILE [USE_TLS=true|false]"
    echo ""
    echo "Parameters:"
    echo "  USERNAME          MongoDB username"
    echo "  PASSWORD          MongoDB password"
    echo "  REPLICA_SET_URI   MongoDB replica set URI (e.g., 'mongodb://rs0/test?replicaSet=rs')"
    echo "  PORT              MongoDB port"
    echo "  FOLDER_PATH       Path to the backup folder"
    echo "  DATABASE_LIST_FILE Path to the text file containing the list of databases"
    echo "  USE_TLS           Optional, enable (true) or disable (false) TLS. Default is true."
    echo ""
    echo "Example:"
    echo "  ./script.sh myuser mypassword 'mongodb://rs0/test?replicaSet=rs' 27017 /backups/ databases.txt false"
}

# Check if the correct number of arguments are provided or if help is requested
if [ "$#" -lt 6 ] || [ "$#" -gt 7 ] || [ "$1" = "--help" ] || [ "$1" = "-h" ]; then
    display_usage
    exit 1
fi

# Assign input parameters
username="$1"
password="$2"
replica_set_uri="$3"
port="$4"
folder_path="$5"
database_list_file="$6"
use_tls="${7:-true}"  # Default to true if not provided

# Check if the database list file exists
if [ ! -f "${database_list_file}" ]; then
    echo "Error: Database list file not found."
    exit 1
fi

# Create a folder with today's date
today=$(date +'%Y-%m-%d')
backup_folder="${folder_path}/${today}"
mkdir -p "${backup_folder}"

# Function to check if a database exists
database_exists() {
    local db="$1"
    tls_flag=""
    if [ "${use_tls}" = "true" ]; then
        tls_flag="--tls"
    fi
    mongo "${tls_flag}" --quiet "${replica_set_uri}/${db}" --username "${username}" --password "${password}" --eval "printjson(db.runCommand({ ping: 1 }).ok)" | grep -q true
}

# Log file for errors
log_file="${folder_path}/mongodump_errors_$(date +'%Y-%m-%d').log"
touch "${log_file}"

# Read the list of databases from the file and loop through each database
while IFS= read -r db; do
    if database_exists "${db}"; then
        echo "Processing database: ${db}"

        # Get collections in the database and print them
        tls_flag=""
        if [ "${use_tls}" = "true" ]; then
            tls_flag="--tls"
        fi
        collections=$(mongo "${tls_flag}" --quiet "${replica_set_uri}/${db}" --username "${username}" --password "${password}" --eval "db.getCollectionNames().join('\n')")
        echo "Collections:"
        echo "${collections}" | while read collection; do
            echo "- ${collection}"
        done

        # Perform mongodump for this database
        tls_flag=""
        if [ "${use_tls}" = "true" ]; then
            tls_flag="--tls"
        fi
        mongodump "${tls_flag}" --uri="mongodb://${username}:${password}@${replica_set_uri}/${db}" --out="${backup_folder}"
    else
        echo "Error: Database '${db}' does not exist." >> "${log_file}"
        echo "Continuing with the next database..."
    fi
done < "${database_list_file}"
```

You can run this script from the command line, providing the required parameters as arguments:

```
chmod +x script.sh
./script.sh USERNAME PASSWORD "REPLICA_SET_URI" PORT FOLDER_PATH DATABASE_LIST_FILE.txt [USE_TLS=true|false]
```

If you run the script with incorrect parameters or with the `--help` or `-h` flag, it will display usage instructions:

```
./script.sh --help
```

Make sure to replace `REPLICA_SET_URI` with your actual replica set URI, such as `"mongodb://rs0/test?replicaSet=rs"`. Also, replace `DATABASE_LIST_FILE.txt` with the path to your text file containing the list of databases, one database name per line.

The optional `USE_TLS` parameter can be set to `true` (default) or `false` to enable or disable TLS, respectively.

This updated script checks if a database exists before processing it. If the database doesn't exist, it logs an error message in a log file named `mongodump_errors_YYYY-MM-DD.log` in the specified folder path and continues with the next database in the list.
