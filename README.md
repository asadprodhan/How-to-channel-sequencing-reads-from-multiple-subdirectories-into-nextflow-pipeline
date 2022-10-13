# **How to channel sequencing reads from multiple subdirectories into nextflow pipeline** <br />

Sequencing reads from a multiplexed library are binned into different barcode subdirectories when demultiplexed. As a pre-analysis data processing, the reads are concatenated barcode-wise and labelled with actual sample names before subjecting them to Bioinformatics Workflows. This pre-analysis preparation can be done manually or by using a bash script like bellow:


```
#!/bin/bash

#metadata
metadata=./*.csv
#
Red="$(tput setaf 1)"
Green="$(tput setaf 2)"
Bold=$(tput bold)
reset=`tput sgr0` # turns off all atribute
while IFS=, read -r field1 field2  

do  
    echo "${Red}${Bold}Processing ${reset}: "${field1}"" 
    echo ""
    echo "Renaming "${field1}" directory as "${field2}"" 
    mv "${field1}" "${field2}" 
    echo "Concatenating "${field2}" reads"
    cd "${field2}" &&
    cat *fastq* > "${field2}".fastq
    echo "Moving "${field2}".fastq into home directory"
    mv "${field2}".fastq ../
    cd "../"
    echo "${Green}${Bold}Completed ${reset}: ${field1}"
    echo ""
done < ${metadata}

```


What if our workflow starts from the barcode subdirectories, contanetanes the reads, labels them, and feeds into the downstream analyses. This will make our workflow fully automated from the start to the end.


Here, I demonstrate how it can be done using Nextflow workflow manager.


## **Requirements**


A csv file that has barcode names in Column 1 and corresponding sample names in Column 2:


```
barcode01	sampleA
barcode02	sampleB
barcode03	sampleC
barcode04	sampleD
barcode05	sampleE
barcode06	sampleF
barcode07	sampleG
barcode08	sampleH
barcode09	sampleI
barcode10	sampleJ
barcode11	sampleK
barcode12	sampleL

```



## **A Nextflow script**


This script contanetanes the reads barcode-wise, labels them taking information from the above metadata, and feeds them into the downstream fastqc analysis. More analyses can be added on.



```
#!/usr/bin/env nextflow

params.metadata = './metadata.csv'
params.outdir = './results'

process concat_reads {

    tag { sample_name }

    publishDir "${params.outdir}/concat_reads", mode: 'copy'

    input:
    tuple val(sample_name), path(fastq_files)

    output:
    tuple val(sample_name), path("${sample_name}.${extn}")

    script:
    if( fastq_files.every { it.name.endsWith('.fastq.gz') } )
        extn = 'fastq.gz'
    else if( fastq_files.every { it.name.endsWith('.fastq') } )
        extn = 'fastq'
    else
        error "Concatentation of mixed filetypes is unsupported"

    """
    cat ${fastq_files} > "${sample_name}.${extn}"
    """
}

process fastqc {

    tag { sample_name }

    publishDir "${params.outdir}/fastqc", mode: 'copy'

    cpus 18

    input:
    tuple val(sample_name), path(fastq)

    output:
    tuple val(sample_name), path("${fastq.simpleName}_fastqc.html")

    """
    fastqc $fastq > ${fastq.simpleName}_fastqc.html
    """
}

workflow {

    fastq_extns = [ '.fastq', '.fastq.gz' ]

    Channel.fromPath( params.metadata )
        | splitCsv()
        | map { dir, sample_name ->

            all_files = file(dir).listFiles()

            fastq_files = all_files.findAll { fn ->
                fastq_extns.find { fn.name.endsWith( it ) }
            }

            tuple( sample_name, fastq_files )
        }
        | concat_reads
        | fastqc
}

```



## **A Nextflow config file**


The config file provides the containerised software to execulte the fastqc analysis. This avoids installing the fastqc software in the local computer. For concatenation step, no container i.e. additional software is needed as ‘cat’ is a standard Linux utility.



```

params.repdir = './reports'
trace {
enabled = true
fields = 'process,task_id,hash,name,attempt,status,exit,submit,start,complete,duration,realtime,cpus,%cpu,disk,memory,%mem,rss,vmem,rchar,wchar,script,workdir'
file = "$params.repdir/consumption.tsv"
sep = '\t'
}

timeline {
  enabled = true
  file = "$params.repdir/timeline.html"
}

report {
  enabled = true
  file = "$params.repdir/run_id.html"
}

process {
    withName:fastqc      {container = 'quay.io/biocontainers/fastqc:0.11.9--hdfd78af_1'}
    
}

docker {
    enabled = true
    temp = 'auto'
}
```



## **How to carry out the workflow**



- Keep the metadata.csv file, nextflow script and the config file in the same directory where the barcode sub-directories are. 


- Then run the script as follows:


```
nextflow nextflow.nf
```



## **Potential error:**


> You may require specifying DSL2 in the config file as follows:



```
nextflow.enable.dsl=2

```

