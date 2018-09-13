# DeepVariant whole genome case study

In this case study we describe applying DeepVariant to a real WGS sample.

We provide some guidelines on the computational resources needed for each step.
And finally we assess the quality of the DeepVariant variant calls with
`hap.py`.

NOTE: This case study demonstrates an example of how to run DeepVariant
end-to-end on one machine. This might not be the fastest or cheapest
configuration for your needs. For more scalable execution of DeepVariant see the
[cost- and speed-optimized, Docker-based
pipelines](https://cloud.google.com/genomics/deepvariant) created for Google
Cloud Platform.

Consult this [document](deepvariant-details.md) for more information about using
GPUs or reading BAM files from Google Cloud Storage (GCS) directly.

## Update since r0.7 : using docker instead of copying binaries.

Starting from the 0.7 release, we use docker to run the binaries instead of
copying binaries to local machines first. You can still read about the previous
approach in
[the Case Study in r0.6](https://github.com/google/deepvariant/blob/r0.6/docs/deepvariant-case-study.md).

We recognize that there might be some overhead of using docker run. But using
docker makes this case study easier to generalize to different versions of Linux
systems. For example, we have verified that you can use docker to run
DeepVariant on other Linux systems such as CentOS 7.

## Request a machine

You can run this exercise on any sufficiently capable machine. As a concrete but
simple example of how we performed this study, we used 64-core (vCPU) machine
with 128GiB of memory and no GPU, on the Google Cloud Platform.

We used a command like this to allocate it:

```shell
gcloud beta compute instances create "${USER}-deepvariant-casestudy"  \
--scopes "compute-rw,storage-full,cloud-platform" \
--image-family "ubuntu-1604-lts" \
--image-project "ubuntu-os-cloud" \
--machine-type "custom-64-131072" \
--boot-disk-size "300" \
--boot-disk-type "pd-ssd" \
--zone "us-west1-b"
```

Once the machine is ready, ssh into it:

```
gcloud compute ssh "${USER}-deepvariant-casestudy" --zone "us-west1-b"
```

NOTE: Having an instance up and running could cost you. Remember to delete the
instances you're not using. You can find the instances at:
https://console.cloud.google.com/compute/instances?project=YOUR_PROJECT

## Preliminaries

Set a number of shell variables, to make what follows easier to read.

```bash
BASE="${HOME}/case-study"
BUCKET="gs://deepvariant"
BIN_VERSION="0.7.0"
MODEL_VERSION="0.7.0"

MODEL_BUCKET="${BUCKET}/models/DeepVariant/${MODEL_VERSION}/DeepVariant-inception_v3-${MODEL_VERSION}+data-wgs_standard"
DATA_BUCKET="${BUCKET}/case-study-testdata"

INPUT_DIR="${BASE}/input"
MODELS_DIR="${INPUT_DIR}/models"
MODEL="${MODELS_DIR}/model.ckpt"
DATA_DIR="${INPUT_DIR}/data"
REF="${DATA_DIR}/hs37d5.fa.gz"
BAM="${DATA_DIR}/HG002_NIST_150bp_50x.bam"
TRUTH_VCF="${DATA_DIR}/HG002_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-22_v.3.3.2_highconf_triophased.vcf.gz"
TRUTH_BED="${DATA_DIR}/HG002_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-22_v.3.3.2_highconf_noinconsistent.bed"

N_SHARDS="64"

OUTPUT_DIR="${BASE}/output"
EXAMPLES="${OUTPUT_DIR}/HG002.examples.tfrecord@${N_SHARDS}.gz"
GVCF_TFRECORDS="${OUTPUT_DIR}/HG002.gvcf.tfrecord@${N_SHARDS}.gz"
CALL_VARIANTS_OUTPUT="${OUTPUT_DIR}/HG002.cvo.tfrecord.gz"
OUTPUT_VCF="${OUTPUT_DIR}/HG002.output.vcf.gz"
OUTPUT_GVCF="${OUTPUT_DIR}/HG002.output.g.vcf.gz"
LOG_DIR="${OUTPUT_DIR}/logs"
```

## Create local directory structure

```bash
mkdir -p "${OUTPUT_DIR}"
mkdir -p "${DATA_DIR}"
mkdir -p "${MODELS_DIR}"
mkdir -p "${LOG_DIR}"
```

## Download extra packages

There are some extra programs we will need.

We are going to use [GNU Parallel](https://www.gnu.org/software/parallel/) to
run `make_examples`. We are going to install `samtools` and `docker.io` to help
do some analysis at the end.

```bash
sudo apt-get -y update
sudo apt-get -y install parallel
sudo apt-get -y install samtools
sudo apt-get -y install docker.io
```

<<<<<<< HEAD
## Download binaries, models, and test data

### Binaries

Copy our binaries from the cloud bucket.

```bash
time gsutil -m cp -r -P "${BIN_BUCKET}/*" "${BIN_DIR}"
```

This step should be very fast - it took us about 6 seconds when we tested.

Now, we need to install all prerequisites on the machine. Run this command:

```bash
cd "${BIN_DIR}"; time bash run-prereq.sh; cd -
```

In our test run it took about 1 min.
||||||| merged common ancestors
## Download binaries, models, and test data

### Binaries

Copy our binaries from the cloud bucket.

```bash
time gsutil -m cp -r "${BIN_BUCKET}/*" "${BIN_DIR}"
```

This step should be very fast - it took us about 6 seconds when we tested.

Now, we need to install all prerequisites on the machine. Run this command:

```bash
cd "${BIN_DIR}"; time bash run-prereq.sh; cd -
```

In our test run it took about 1 min.
=======
## Download models, and test data
>>>>>>> 3c43de4541c45673e30d14daef742fca68fdf69b

### Models

Copy the model files to your local disk.

```bash
time gsutil -m cp -r "${MODEL_BUCKET}/*" "${MODELS_DIR}"
```

This step should be really fast. It took us about 5 seconds.

### Test data

Copy the input files you need to your local disk from our gs:// bucket.

The original source of these files are:

1.  BAM file: HG002_NIST_150bp_50x.bam

    The original FASTQ file comes from the [PrecisionFDA Truth
    Challenge](https://precision.fda.gov/challenges/truth/). I ran it through
    [PrecisionFDA's BWA-MEM
    app](https://precision.fda.gov/apps/app-BpF9YGQ0bxjbjk6Fx1F0KJF0) with
    default setting, and then got the HG002_NIST_150bp_50x.bam file as output.
    The FASTQ files are originally from the [Genome in a Bottle
    Consortium](http://jimb.stanford.edu/giab-resources/). For more information,
    see this [Scientific Data
    paper](https://www.nature.com/articles/sdata201625).

1.  FASTA file: hs37d5.fa.gz

    The original file came from:
    [ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/phase2_reference_assembly_sequence](ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/phase2_reference_assembly_sequence).
    Because DeepVariant requires bgzip files, we had to unzip and bgzip it, and
    create corresponding index files.

1.  Truth VCF and BED

    These come from NIST, as part of the [Genome in a Bottle
    project](http://jimb.stanford.edu/giab/). They are downloaded from
    [ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/release/AshkenazimTrio/HG002_NA24385_son/NISTv3.3.2/GRCh37/](ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/release/AshkenazimTrio/HG002_NA24385_son/NISTv3.3.2/GRCh37/)

You can simply run the command below to get all the data you need for this case
study.

```bash
time gsutil -m cp -r "${DATA_BUCKET}/*" "${DATA_DIR}"
```

It took us about 15min to copy the files.

## Run `make_examples`

First, to set up,

```
sudo docker pull gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}"
```

Because the informational log messages (written to stderr) are voluminous, and
because this takes a long time to finish, we will redirect all the output
(stdout and stderr) to a file.

```bash
( time seq 0 $((N_SHARDS-1)) | \
  parallel --halt 2 --joblog "${LOG_DIR}/log" --res "${LOG_DIR}" \
    sudo docker run \
      -v /home/${USER}:/home/${USER} \
      gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
      /opt/deepvariant/bin/make_examples \
      --mode calling \
      --ref "${REF}" \
      --reads "${BAM}" \
      --examples "${EXAMPLES}" \
      --gvcf "${GVCF_TFRECORDS}" \
      --task {} \
) >"${LOG_DIR}/make_examples.log" 2>&1
```

This will take several hours to run.

## Run `call_variants`

There are different ways to run `call_variants`. In this case study, we ran just
one `call_variants` job. Here's the command that we used:

```bash
( time sudo docker run \
    -v /home/${USER}:/home/${USER} \
    gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
    /opt/deepvariant/bin/call_variants \
    --outfile "${CALL_VARIANTS_OUTPUT}" \
    --examples "${EXAMPLES}" \
    --checkpoint "${MODEL}"
) >"${LOG_DIR}/call_variants.log" 2>&1
```

This will start one process of `call_variants` and read all 64 shards of
`HG002.examples.tfrecord@64.gz` as input, but output to one single output file
`HG002.cvo.tfrecord.gz`.

We add `--batch_size 32` here because we noticed a significant slowdown after
the Meltdown/Spectre CPU patch if we continue to use the default batch_size
(512).

We noticed that running multiple `call_variants` on the same machine didn't seem
to save the overall time, because each of the call_variants slowed down when
multiple are running on the same machine.

So if you care about finishing the end-to-end run faster, you could request 64
machines (for example, `n1-standard-8` machines) and run each shard as input
separately, and output to corresponding output shards. For example, the first
machine will run this command:

```shell
time sudo docker run \
  -v /home/${USER}:/home/${USER} \
  gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
  /opt/deepvariant/bin/call_variants \
  --outfile=${OUTPUT_DIR}/HG002.cvo.tfrecord-00000-of-00064.gz \
  --examples=${OUTPUT_DIR}/HG002.examples.tfrecord-00000-of-00064.gz \
  --checkpoint="${MODEL}"
```

And the rest will process 00001 to 00063. You can also use tools like
[dsub](https://cloud.google.com/genomics/v1alpha2/dsub). In this case study we
only report the runtime on a 1-machine example.

## Run `postprocess_variants`

```bash
( time sudo docker run \
    -v /home/${USER}:/home/${USER} \
    gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
    /opt/deepvariant/bin/postprocess_variants \
    --ref "${REF}" \
    --infile "${CALL_VARIANTS_OUTPUT}" \
    --outfile "${OUTPUT_VCF}"
) >"${LOG_DIR}/postprocess_variants.log" 2>&1
```

Because this step is single-process, single-thread, if you're orchestrating a
more complicated running pipeline, you might want to request a machine with
fewer cores for this step.

If you want to create a gVCF output file, two additional flags must be passed to
the postprocess\_variants step, so the full call would look instead like:

```bash
( time sudo docker run \
    -v /home/${USER}:/home/${USER} \
    gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
    /opt/deepvariant/bin/postprocess_variants \
    --ref "${REF}" \
    --infile "${CALL_VARIANTS_OUTPUT}" \
    --outfile "${OUTPUT_VCF}" \
    --nonvariant_site_tfrecord_path "${GVCF_TFRECORDS}" \
    --gvcf_outfile "${OUTPUT_GVCF}"
) >"${LOG_DIR}/postprocess_variants.withGVCF.log" 2>&1
```

## Resources used by each step

Step                               | wall time
---------------------------------- | ---------------
`make_examples`                    | 1h 45m 46s
`call_variants`                    | 3h 25m 38s
`postprocess_variants` (no gVCF)   | 21m 33s
`postprocess_variants` (with gVCF) | 55m 47s
total time (single machine)        | 5h 33m - 6h 07m

## Variant call quality

Here we use the `hap.py`
([https://github.com/Illumina/hap.py](https://github.com/Illumina/hap.py))
program from Illumina to evaluate the resulting vcf file. This serves as a check
to ensure the three DeepVariant commands ran correctly and produced high-quality
results.

```bash
UNCOMPRESSED_REF="${OUTPUT_DIR}/hs37d5.fa"

# hap.py cannot read the compressed fa, so uncompress
# into a writable directory and index it.
zcat <"${REF}" >"${UNCOMPRESSED_REF}"
samtools faidx "${UNCOMPRESSED_REF}"

sudo docker pull pkrusche/hap.py
sudo docker run -it \
-v "${DATA_DIR}:${DATA_DIR}" \
-v "${OUTPUT_DIR}:${OUTPUT_DIR}" \
pkrusche/hap.py /opt/hap.py/bin/hap.py \
  "${TRUTH_VCF}" \
  "${OUTPUT_VCF}" \
  -f "${TRUTH_BED}" \
  -r "${UNCOMPRESSED_REF}" \
  -o "${OUTPUT_DIR}/happy.output" \
  --engine=vcfeval
```

Type  | # FN | # FP | Recall   | Precision | F1\_Score
----- | ---- | ---- | -------- | --------- | ---------
INDEL | 1431 | 916  | 0.996921 | 0.998106  | 0.997513
SNP   | 1329 | 746  | 0.999564 | 0.999755  | 0.999660
