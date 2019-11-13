# BLAST-DIAMOND-Cloud-VM
Instructions for running BLAST and DIAMOND on a GCP VM instance


### 1) Set up your GCP account and VM instance following these instuctions
https://github.com/ncbi/blast_plus_docs/blob/master/README.md#set-up-your-gcp-account-and-create-a-vm-for-analysis


* When creating your virtual machine, use the following settings:
    * Machine Type: **16 vCPU, 104 GB memory, n1-highmem-16**
    * Boot Disk: Click "Change" and select **Ubuntu 18.04 LTS**, change the "Boot disk size" to **200 GB** Standard persistent disk, and click "Select." 
* Enable HTTP and HTTPs access
* Expand the Management, security, disks, networking, sole tenancy section.
* Expand the networking section
* 'Edit' the network interfaces section. You'll see this:

* Open the External IP dropdown and select "Create IP Address"
    * name the IP address anything you want. This can later be used as a static IP address for your project.
    * click "Done" when completed.
* In the **Automation** section, paste the following script:
```
#TODO
```

Continue through the tutorial to open the GCP command shell

### 2) Install Docker following the same tutorial

https://github.com/ncbi/blast_plus_docs/blob/master/README.md#step-1-install-docker

Make sure you exit and ssh back in for the installation to take effect! The *optional* sections following this instruction offer helpful information about using docker.

### 3) Create BLAST and DIAMOND containers from DockerHub
```
docker run ncbi/blast
docker run buchfink/diamond
docker images
```

### 4) Set up file structure
```
cd ; mkdir blastdb tempfastabd diamonddb queries fasta blastresults diamondresults blastdb_custom
```

### 5) Access BLAST databases

To see the available databases:
```
docker run --rm ncbi/blast update_blastdb.pl --showall pretty --source gcp
```

**Note: make sure you don't try to download a database larger than the memory size you chose for yor VM**

To download your chosen database:

```
docker run --rm -v $HOME/blastdb:/blast/blastdb:rw -w /blast/blastdb ncbi/blast update_blastdb.pl --source gcp [DATABASE]
```

where [DATABASE] is the name of one of the databases above that you want to download

### 6) Convert BLAST database to DIAMOND database

```
docker run --rm -v $HOME/blastdb:/blast/blastdb:rw -v $HOME/tempfastadb:/blast/tempfastadb:rw -w /blast/blastdb ncbi/blast blastdbcmd -entry all -db [DATABASE] > ../tempfastadb/fasta_[DATABASE].fa

docker run --rm -v $HOME/tempfastadb:/diamond/tempfastadb -v $HOME/diamonddb:/blast/diamonddb:rw -w /diamond/diamonddb  buchfink/diamond diamond makedb -d [DATABASE] --in fastadb.fa > ../diamonddb/[DATABASE].dmdb
```

Now your databases are ready to use!

### 7) Retreive query sequences

If you'd like to use a sequence from an existing NCBI database, you can access it using blast's efetch command. For example:

```
## Retrieve query sequence
docker run --rm ncbi/blast efetch -db protein -format fasta -id P01349 > queries/P01349.fsa
```

However, we are going to use some sample sequences stored on our local machine!

### 8) Upload files from local machine

First, install Google Cloud SDK in your terminal, following these instructions for your OS:

https://cloud.google.com/sdk/docs/quickstarts

Follow the prompts to install the SDK and make sure to connect it to the project where your VM is located and the zone you selected when creating your VM.

You can ssh into your VM from your local machine using:
```
gcloud beta compute ssh [USER@]INSTANCE

# For example, using my gmail account name and the name of my instance:
# gcloud beta compute ssh kellyfinke1201@diamond-instance
```
For more info: https://cloud.google.com/sdk/gcloud/reference/beta/compute/ssh

The Google Cloud SDK installation includes PuTTy, which is used to ssh into the VM.

Back in your terminal, you can transfer your query files to your virtual machine using:

https://cloud.google.com/compute/docs/instances/transfer-files

gcloud compute scp samples.fasta kellyfinke1201@diamond-instance:~   

```
gcloud beta compute scp [PATH TO FILE] [USER@]INSTANCE:DESTINATION

# For example, using my gmail account name and the name of my instance:
# gcloud compute scp samples.fasta kellyfinke1201@diamond-instance:queries/samples.fasta
```

You can ssh back in to the VM to check if your files are in the correct place

### 9) Run BLAST and DIAMOND
