---
id: snapshot-instructions-heimdall-bor
title: Heimdall and Bor Snapshots
sidebar_label: Heimdall & Bor Snapshots
description: Heimdall and Bor snapshot Instructions for faster syncing.
keywords:
  - docs
  - matic
  - polygon
  - binary
  - node
  - validator
  - sentry
  - heimdall
  - bor
  - snapshots
image: https://wiki.polygon.technology/img/polygon-wiki.png
---

import useBaseUrl from '@docusaurus/useBaseUrl';

When setting up a new sentry, validator, or full node server, it is recommended that you use snapshots for faster syncing without having to sync over the network. Using snapshots will save you several days for both Heimdall and Bor.

:::tip

For the latest snapshot, please visit [<ins>Polygon Chains Snapshots</ins>](https://snapshot.polygon.technology/).

:::

## Heimdall/Bor Snapshots

To begin, ensure that your node environment meets the **prerequisites** outlined [here](https://wiki.polygon.technology/docs/operate/full-node-binaries/). Before starting any services, execute the shell script provided below. This script will download and extract the snapshot data, which allows for faster bootstrapping. In our example, we will be using an Ubuntu Linux m5d.4xlarge machine with an 8TB block device attached.
To transfer the correct chaindata to your disk, follow these steps:

- When prompted, specify the network (mainnet or mumbai) and the client type (heimdall or bor).
> The script will automatically handle the download and extraction phases, optimizing disk space by deleting already extracted files.
- Consider using a Screen session to prevent accidental interruptions during the chaindata download and extraction process.

```
#!/bin/bash

function validate_network() {
  if [[ "$1" != "mainnet" && "$1" != "mumbai" ]]; then
    echo "Invalid network input. Please enter 'mainnet' or 'mumbai'."
    exit 1
  fi
}

function validate_client() {
  if [[ "$1" != "heimdall" && "$1" != "bor" ]]; then
    echo "Invalid client input. Please enter 'heimdall' or 'bor'."
    exit 1
  fi
}

# ask user for network and client type
read -p "PoSV1 Network (mainnet/mumbai): " network_input
validate_network "$network_input"
read -p "Client Type (heimdall/bor): " client_input
validate_client "$client_input"
read -p "Directory to Download/Extract: " extract_dir_input

# set default values if user input is blank
network=${network_input:-mumbai}
client=${client_input:-heimdall}
extract_dir=${extract_dir_input:-"${client}_extract"}

# install dependencies and cursor to extract directory
sudo apt-get update -y
sudo apt-get install -y zstd pv aria2
mkdir -p "$extract_dir"
cd "$extract_dir"

# download compiled incremental snapshot files list
aria2c -x6 -s6 "https://snapshot-download.polygon.technology/$client-$network-incremental-compiled-files.txt"

# download all incremental files, includes automatic checksum verification per increment
aria2c -x6 -s6 -i $client-$network-incremental-compiled-files.txt

# helper method to extract all files and delete already-extracted download data to minimize disk use
function extract_files() {
    compiled_files=$1
    while read -r line; do
        if [[ "$line" == checksum* ]]; then
            continue
        fi
        filename=`echo $line | awk -F/ '{print $NF}'`
        if echo "$filename" | grep -q "bulk"; then
            pv $filename | tar -I zstd -xf - -C . && rm $filename
        else
            pv $filename | tar -I zstd -xf - -C . --strip-components=3 && rm $filename
        fi
    done < $compiled_files
}

# execute final data extraction step
extract_files $client-$network-incremental-compiled-files.txt
```

**Note:** If experiencing intermittent aria2c download errors, try reducing concurrency as exampled here:
```
aria2c -c -m 0 -x6 -s6 -i heimdall-$network-incremental-compiled-files.txt --max-concurrent-downloads=1
```

Once the extraction is complete, ensure that you update the datadir configuration of your Heimdall or Bor client to point to the path where the extracted data is located.
This ensures that the systemd services can correctly register the snapshot data when the client starts. 
If you wish to preserve the default client configuration settings, you can use symbolic links (symlinks).

For example, let's say you have mounted your block device at `~/snapshots` and have downloaded and extracted the chaindata
for Heimdall into the directory `heimdall_extract`, and for Bor into the directory `bor_extract`. To ensure proper registration
of the extracted data when starting the Heimdall or Bor systemd services, you can use the following sample commands:
```
# remove any existing datadirs for heimdall and bor
rm -rf /var/lib/heimdall/data
rm -rf /var/lib/bor/chaindata

# rename and setup symlinks to match default client datadir configs
mv ~/snapshots/heimdall_extract ~/snapshots/data
mv ~/snapshots/bor_extract ~/snapshots/chaindata
sudo ln -s ~/snapshots/data /var/lib/heimdall
sudo ln -s ~/snapshots/chaindata /var/lib/bor

# bring up clients with all snapshot data properly registered
sudo service heimdalld start
# wait for heimdall to fully sync then start bor
sudo service bor start
```

## Recommended disk size guidance

**MUMBAI:**

| metric | calculation breakdown | value |
| ------ | --------------------- | ----------- |
| approx. compressed total | 250 GB (bor) + 35 GB (heimdall) | 285 GB |
| approx. data growth daily | 10 GB (bor) + .5 GB (heimdall) | 10.5 GB |
| approx. total extracted size | 350 GB (bor) + 50 GB (heimdall) | 400 GB |
| suggested disk size (2.5x buffer) | 400 GB * 2.5 (natural chain growth) | 1 TB | 

**MAINNET:**

| metric | calculation breakdown | value |
| ------ | --------------------- | ----------- |
| approx. compressed total | 1500 GB (bor) + 225 GB (heimdall) | 1725 GB |
| approx. data growth daily | 100 GB (bor) + 5 GB (heimdall) | 105 GB |
| approx. total extracted size | 2.1 TB (bor) + 300 GB (heimdall) | 2.4 TB |
| suggested disk size (2.5x buffer) | 2.4 TB * 2.5 (natural chain growth) | 6 TB |
