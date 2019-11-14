# BLAST-DIAMOND-Cloud-VM
Instructions for running BLAST and DIAMOND on a GCP VM instance


### 1) Set up your GCP account and VM instance following these instuctions
https://github.com/ncbi/blast_plus_docs/blob/master/README.md#set-up-your-gcp-account-and-create-a-vm-for-analysis


* When creating your virtual machine, use the following settings:
    * Machine Type: **16 vCPU, 104 GB memory, n1-highmem-16*
      * if this exceeds your CPU limit (you'd get a warning when attempting to create the instance, you can instead use **n1-highmem-8 (8 vCPU, 52 GB memory)**, but keep in mind you will be limited in the size of database you can download
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

### 3) Set up file structure
```
cd ; mkdir blastdb tempfastadb diamonddb queries fasta blastresults diamondresults blastdb_custom
```

### 4) Create BLAST and DIAMOND containers from DockerHub
```
docker run ncbi/blast
docker run buchfink/diamond
docker images
```

docker run \
    -v $HOME/blastdb:/blast/blastdb:rw \
    -v $HOME/fasta:/blast/fasta:ro \
    -v $HOME/tempfastadb:/blast/tempfastadb:rw \
    -v $HOME/queries:/blast/queries:rw \
    -v $HOME/blastresults:/blast/blastresults:rw \
    -t -d --name blast ncbi/blast

blast can be opened and running in the background, but diamond does not have support for that, so we will have to create new image each time we want to run a diamond command, however, that will not be costly as diamond images are very small and all of the databases will be stored in our VM not in the container.

docker run \
    -v $HOME/diamonddb:/diamond/diamonddb:rw \
    -v $HOME/tempfastadb:/diamond/tempfastadb:rw \
    -v $HOME/queries:/diamond/queries:rw \
    -v $HOME/diamondresults:/diamond/diamondresults:rw \
    -t -d --name diamond buchfink/diamond


### 5) Access BLAST databases

To see the available databases:
```
docker run --rm ncbi/blast update_blastdb.pl --showall pretty --source gcp

docker exec blast update_blastdb.pl --showall pretty --source gcp
```

**Note: make sure you don't try to download a database larger than the memory size you chose for yor VM**

To download your chosen database:

```
docker run --rm -v $HOME/blastdb:/blast/blastdb:rw -w /blast/blastdb ncbi/blast update_blastdb.pl --source gcp [DATABASE]

docker exec -w /blast/blastdb blast update_blastdb.pl --source gcp [DATABASE] 
```

where [DATABASE] is the name of one of the databases above that you want to download

Now, if you cd into the blastdb folder, you should see your downloaded database

### 6) Convert BLAST database to DIAMOND database

```
docker run --rm -v $HOME/blastdb:/blast/blastdb:rw -v $HOME/tempfastadb:/blast/tempfastadb:rw -w /blast/blastdb ncbi/blast blastdbcmd -entry all -db [DATABASE] > ../tempfastadb/fasta_[DATABASE].fa

docker run --rm -v $HOME/tempfastadb:/diamond/tempfastadb -v $HOME/diamonddb:/blast/diamonddb:rw -w /diamond/diamonddb  buchfink/diamond diamond makedb -d [DATABASE] --in fastadb.fa > ../diamonddb/[DATABASE].dmnd
```

```
docker exec  blast blastdbcmd -entry all -db pdb_v5 > tempfastadb/fasta_pdb_v5.fa

docker run --rm -v $HOME/tempfastadb:/diamond/tempfastadb -v $HOME/diamonddb:/diamond/diamonddb:rw  buchfink/diamond diamon
d makedb -d pdb_v5 --in /diamond/tempfastadb/fasta_pdb_v5.fa > diamonddb/pdb_v5.dmnd

docker exec blast blastdbcmd -entry all -db [DATABASE] > tempfastadb/fasta_[DATABASE].fa

docker run --rm -v $HOME/tempfastadb:/diamond/tempfastadb -v $HOME/diamonddb:/diamond/diamonddb:rw  buchfink/diamond diamon
d makedb -d [DATABASE] --in /diamond/tempfastadb/fasta_[DATABASE].fa > diamonddb/[DATABASE].dmnd
```

Now your databases are ready to use!

### 7) Retreive query sequences

If you'd like to use a sequence from an existing NCBI database, you can access it using blast's efetch command. For example:

```
## Retrieve query sequence
docker exec blast efetch -db protein -format fasta -id P01349 > queries/P01349.fsa
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
# gcloud beta compute ssh kellyfinke1201@blast-diamond-instance
```
For more info: https://cloud.google.com/sdk/gcloud/reference/beta/compute/ssh

The Google Cloud SDK installation includes PuTTy, which is used to ssh into the VM. You can type `exit` to exit

Back in your terminal, you can transfer your query files to your virtual machine using:

https://cloud.google.com/compute/docs/instances/transfer-files

gcloud compute scp samples.fasta kellyfinke1201@diamond-instance:~   

```
gcloud beta compute scp [PATH TO FILE] [USER@]INSTANCE:DESTINATION

# For example, using my gmail account name and the name of my instance (From the directory where protein_queries.fasta is stored, though you could also just give the path to you sequences):
# gcloud compute scp protein_queries.fasta kellyfinke1201@blast-diamond-instance:queries/protein_queries.fasta
```

You can ssh back in to the VM to check if your files are in the correct place

### 9) Run BLAST and DIAMOND

You can run the following from either your local machine via ssh or from the GCP command shell:

```
# Run BLAST alignment
# Where [QUERY FILE] contains the sequences in .fasta, .fa, .faa format and [OUT] is the name you want for the output

docker exec blast blastp -query /blast/queries/[QUERY FILE] -db [DATABASE] -out /blast/blastresults/[OUT].out

# Run DIAMOND alignment

docker run \
    -v $HOME/queries:/diamond/queries:ro \
    -v $HOME/diamondresults:/diamond/diamondresults:rw \
    buchfink/diamond diamond blastp -d pdb_v5 -q queries/protein_queries.fasta -o diamond_proteins.m8
```

## Summary: Full Script

Here are all the commands we ran in order (after creating the VM as described in step 1, which we named 'blast-diamond-instance' in the example below). For our example, we are using pdb_v5 as our database and two sample sequences stored in protein_queries.fasta in this repository for you to download for testing.

In the GCP command shell:
```
## Run these commands to install Docker and add non-root users to run Docker
sudo snap install docker
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
exit
```
(Exit and SSH back in for changes to take effect)
```
cd ; mkdir blastdb tempfastadb diamonddb queries fasta blastresults diamondresults blastdb_custom

docker run \
    -v $HOME/blastdb:/blast/blastdb:rw \
    -v $HOME/fasta:/blast/fasta:ro \
    -v $HOME/tempfastadb:/blast/tempfastadb:rw \
    -v $HOME/queries:/blast/queries:rw \
    -v $HOME/blastresults:/blast/blastresults:rw \
    -t -d --name blast ncbi/blast

docker run buchfink/diamond


docker exec blast update_blastdb.pl --showall pretty --source gcp
docker exec -w /blast/blastdb blast update_blastdb.pl --source gcp pdb_v5

docker exec  blast blastdbcmd -entry all -db pdb_v5 > tempfastadb/fasta_pdb_v5.fa

docker run --rm \
   -v $HOME/tempfastadb:/diamond/tempfastadb:ro \
   -v $HOME/diamonddb:/diamond/diamonddb:rw \
   buchfink/diamond \
   diamond makedb -d pdb_v5 --in /diamond/tempfastadb/fasta_pdb_v5.fa > diamonddb/pdb_v5.dmnd
```
Now, from your local machine, cd to the directory where you saved protein_queries.fasta and run the following:
```
# Just for fun! You can ssh into your VM instance ({USERNAME] is your GCP account username): 
gcloud beta compute ssh [USERNAME]@blast-diamond-instance

# To actually transfer the file:
gcloud compute scp protein_queries.fasta [USERNAME]@blast-diamond-instance:queries/protein_queries.fasta
```
You can run the following either through the GCP command shell or through ssh on your local machine (using `gcloud beta compute ssh` as shown above)
```
docker exec blast blastp -query /blast/queries/protein_queries.fasta -db pdb_v5 -out /blast/blastresults/blast_proteins.out

docker run --rm \
   -v $HOME/queries:/diamond/queries:ro \
   -v $HOME/diamonddb:/diamond/diamonddb:rw \
   -v $HOME/diamondresults:/diamond/diamondresults:rw \
   -w /diamond/diamonddb \
   buchfink/diamond \
   diamond blastp -d pdb_v5 -q /diamond/queries/protein_queries.fasta -o /diamond/diamond_proteins.m8
```


When you're done, remember to stop your blast container using `docker stop blast` and to stop your VM instance (or else you'll get charged for all the time you're not using it!)

### 10) Using a startup script
If you want any of this to run automatically when you start up your VM, you can use a startup script:
https://cloud.google.com/compute/docs/startupscript

### 11) Using Service Accounts

If you want your VM to be accessible by others, you can create a service account and attach it to your VM following these instructions: https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances?authuser=1

Then, when using gcloud compute ssh or gcloud compute scp, you simply use the service account name instead of your user name and use the corresponding key (produced when you create the service account) to access the VM.
