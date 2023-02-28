# Training a DeepConsensus Model on a Cloud TPU VM

This tutorial demonstrates how to train DeepConsensus on a Cloud TPU VM.

Training examples can be created with DeepConsensus. See the
[Generate Training Examples](generate_examples.md) for details. For this
tutorial we will point to a pre-generated dataset.

To learn more about Google Cloud TPU VMs, you can read
[this blog post](https://cloud.google.com/blog/products/compute/introducing-cloud-tpu-vms)
or [this user guide](https://cloud.google.com/tpu/docs/users-guide-tpu-vm).

To learn more about pricing, including some free credits for new customers, see:
https://cloud.google.com/tpu/pricing

In this tutorial, we'll assume that you have the permissions and quota set up.
We'll use a specific TPU configuration to run through the tutorial. But you can
choose other TPU configurations.

NOTE: This tutorial has NOT been optimized for cost. Please proceed with care.

## Get a Cloud TPU VM, and a persistent disk attached to it.

I followed the steps on https://cloud.google.com/tpu/docs/setup-persistent-disk.

```bash
PROJECT=<YOUR_PROJECT_NAME>
ZONE=us-central1-c
```


```bash
gcloud compute disks create ${USER}-tpu-disk \
--size 2T  \
--zone ${ZONE} \
--type pd-balanced \
--project ${PROJECT}
```

This disk will still need to be formatted. We will use a small (cheap) VM to
format the disk and copy the data. Then we will attach our persistent disk to
the TPU VM. This way we won't waste TPU time.

Create a small VM:

```bash
gcloud compute instances create "${USER}-disk-setup"   \
--scopes "compute-rw,storage-full,cloud-platform"   \
--maintenance-policy "TERMINATE" \
--image-family "ubuntu-2004-lts" \
--image-project "ubuntu-os-cloud" \
--machine-type "n1-standard-4" \
--zone $ZONE \
--project ${PROJECT}
```

Attach persistent disk:

```bash
gcloud compute instances attach-disk "${USER}-disk-setup" \
--disk "${USER}-tpu-disk" \
--zone $ZONE \
--project ${PROJECT}
```

SSH to the newly created VM: Please note that it takes couple of minutes for the
VM to start up before you can SSH to it.

```bash
gcloud compute ssh "${USER}-disk-setup" \
--zone $ZONE \
--project ${PROJECT}
```

format the disk as described in
https://cloud.google.com/tpu/docs/setup-persistent-disk#setting_up_a_tpu_vm_and_a_persistent_disk:

I ran:

```bash
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
```

to format. If you're using a disk that already has been formatted and maybe even
contains data already, you can skip formatting.

And then:

```bash
sudo mkdir -p /mnt/disks/persist
sudo mount -o discard,defaults /dev/sdb /mnt/disks/persist
sudo chmod a+w /mnt/disks/persist
```

Then, I copied over a dataset:

```bash
BASE_DIR=/mnt/disks/persist/dc_training_examples
mkdir -p ${BASE_DIR}/tf_examples
time gcloud alpha storage cp -R \
gs://brain-genomics-public/research/deepconsensus/training-tutorial/v1.2/* \
${BASE_DIR}/tf_examples
```

NOTE: If you get an error like:
`ERROR: (gcloud.alpha.storage.cp) HTTPError 403:`
`does not have storage.objects.get access to the Google Cloud Storage object.`,
just make sure you run `gcloud init` and `gcloud auth login` with the right
account that has access to the project you're using.

NOTE: This copying step takes a while.

The cp command showed this at the end:

```
  Completed files 2002/2002 | 1.2TiB/1.2TiB | 703.7MiB/s

Average throughput: 735.6MiB/s

real    84m37.041s
user    94m17.018s
sys     69m5.970s
```

Exit the VM and detach persistent disk from temporary VM:

```bash
exit
gcloud compute instances detach-disk "${USER}-disk-setup" \
--disk "${USER}-tpu-disk" \
--zone $ZONE \
--project ${PROJECT}
```

Get a Cloud TPU VM (`--accelerator-type=v2-8` specifies Cloud TPU v2):

```bash
gcloud compute tpus tpu-vm create ${USER}-tpu-name \
--zone=${ZONE} \
--accelerator-type=v2-8 \
--version=tpu-vm-tf-2.9.1 \
--project ${PROJECT} \
--data-disk source=projects/${PROJECT}/zones/${ZONE}/disks/${USER}-tpu-disk,mode=read-write
```

SSH to the newly created TPU VM. Please note that it takes time for VM to boot
up before you can SSH to it.

```bash
gcloud compute tpus tpu-vm ssh ${USER}-tpu-name \
  --zone ${ZONE} --project ${PROJECT}
```

Mount persistent disk:

```bash
sudo mkdir -p /mnt/disks/persist
sudo mount -o discard,defaults /dev/sdb /mnt/disks/persist
sudo chmod a+w /mnt/disks/persist
```

## Install DeepConsensus

```bash
git clone https://github.com/google/deepconsensus.git
```

```
cd deepconsensus
sed -i -e 's|python3 -m pip install --user "intel-tensorflow==2.9.1"||' install.sh
./install.sh
```

The `sed` line is important: we don't want to re-install TensorFlow because that
will mess up the pre-installed TensorFlow version on the Cloud TPU.

Just to confirm that our installation didn't mess up the pre-installed version,
it's good to follow the steps in
https://cloud.google.com/tpu/docs/run-calculation-tensorflow#run_a_simple_example_using_tensorflow
to confirm that we can still run the simple test after installing DeepConsensus.

## Set up variables and files

```bash
BASE_DIR=/mnt/disks/persist/dc_training_examples
export DC_TRAIN_DIR="${HOME}/dc-model"
export TF_EXAMPLES="${BASE_DIR}/tf-examples"  # This should already be there.
export DC_TRAIN_OUTPUT="${DC_TRAIN_DIR}/output"

mkdir -p "${DC_TRAIN_DIR}"
mkdir -p "${DC_TRAIN_OUTPUT}"
```

The path to training examples has to be set in
`deepconsensus/models/model_configs.py` in `_set_custom_data_hparams` function.

For example, if training data is located in
/mnt/disks/persist/dc_training_examples/tf_examples the config will look like
this:

```bash
def _set_custom_data_hparams(params):
  """Updates the given config with values for human data aligned to CCS."""
  params.tf_dataset = ['/mnt/disks/persist/dc_training_examples/tf_examples']
  params.max_passes = 20
  # Note that these are not the actual number of examples. We're setting these
  # numbers so that training will go faster.
  # This is something we might want to improve later because it can be a bit
  # confusing.
  params.n_examples_train = 100_000_000
  params.n_examples_eval = 3_500_000
  # Set this to True if the tf examples contain ccs base quality scores.
  # Option available starting in v1.2.
  params.use_ccs_bq = True
```

It is assumed that after copying training examples the
`/mnt/disks/persist/dc_training_examples/tf_examples` directory contains the
following:

*   train
*   eval
*   test

In addition there is `summary.training.json` file which contains actual
*n_examples_train* and *n_examples_eval* values.

## Launch training script

```bash
export PYTHONPATH=$PWD:$PYTHONPATH
export CONFIG=deepconsensus/models/model_configs.py:transformer_learn_values+custom

time python3 deepconsensus/models/model_train_custom_loop.py \
  --params ${CONFIG} \
  --out_dir ${DC_TRAIN_OUTPUT} \
  --alsologtostderr \
  --tpu=local \
  --tpu_topology=4x4 2>&1 | tee /tmp/dc-tpu.log &
```

`--tpu_topology=4x4` here should work for TPU v2 and v3.

## Optionally launch training from a checkpoint (warm-start training)

Beginning training from an existing model checkpoint will generally speed up
most training sessions. In order to start from a checkpoint add `--checkpoint`
parameter that points to the path + prefix of the checkpoint. DeepConsensus v1.2
checkpoint can be copied from
`gs://brain-genomics-public/research/deepconsensus/models/v1.2/model_checkpoint`
This directory contains 3 files:

```
gs://brain-genomics-public/research/deepconsensus/models/v1.2/model_checkpoint/checkpoint.data-00000-of-00001
gs://brain-genomics-public/research/deepconsensus/models/v1.2/model_checkpoint/checkpoint.index
gs://brain-genomics-public/research/deepconsensus/models/v1.2/model_checkpoint/params.json
```

Copy DeepConsensus checkpoint locally:

```bash
mkdir /mnt/disks/persist/model_checkpoint
gsutil cp gs://brain-genomics-public/research/deepconsensus/models/v1.2/model_checkpoint/* /mnt/disks/persist/model_checkpoint/
```

Add optional `--checkpoint` flag to
`deepconsensus/models/model_train_custom_loop.py`

```bash
export PYTHONPATH=$PWD:$PYTHONPATH
export CONFIG=deepconsensus/models/model_configs.py:transformer_learn_values+custom
time python3 deepconsensus/models/model_train_custom_loop.py \
  --checkpoint /mnt/disks/persist/model_checkpoint/checkpoint \
  --params ${CONFIG} \
  --out_dir ${DC_TRAIN_OUTPUT} \
  --alsologtostderr \
  --tpu=local \
  --tpu_topology=4x4 2>&1 | tee /tmp/dc-tpu.log &
```

## Runtime

In the log, you can find what the batch size was set to. For example:

```
I1026 05:48:32.895524 140202426203200 model_utils.py:271] Per-replica batch-size is 256.
I1026 05:48:32.895847 140202426203200 model_utils.py:280] Global batch size is 8192
```

By default, training will run for 9 epochs. Per-replica batch size and epochs
can be configured by updating the `model_configs.py` file. Global batch size is
scaled based on the TPU topology and number of cores you have available.

The number of steps in an epoch = `<number of examples>` /
`<global_batch_size>`. Once the training is finished the
`$DC_TRAIN_OUTPUT/best_checkpoint.txt` file will contain the best performing
checkpoint. This checkpoint can then be used during inference.

As the command runs, you can check the log file to see the `eval` metrics:

```
$ grep 'eval/' /tmp/dc-tpu.log 
I1026 06:51:25.169192 140202426203200 model_utils.py:486] epoch: 0  step: 427 of 427 metrics: eval/loss= 0.2860963046550751 eval/per_example_accuracy= 0.7721399068832397 eval/per_batch_alignment_identity= 0.9938183426856995 eval/yield_over_ccs= 1.2654321193695068
I1026 07:52:14.438571 140202426203200 model_utils.py:486] epoch: 0  step: 427 of 427 metrics: eval/loss= 0.2881740629673004 eval/per_example_accuracy= 0.7737379670143127 eval/per_batch_alignment_identity= 0.9937282204627991 eval/yield_over_ccs= 1.602739691734314
I1026 08:53:02.877390 140202426203200 model_utils.py:486] epoch: 0  step: 427 of 427 metrics: eval/loss= 0.29722288250923157 eval/per_example_accuracy= 0.782492995262146 eval/per_batch_alignment_identity= 0.9936873912811279 eval/yield_over_ccs= 2.276315689086914
I1026 09:53:51.611985 140202426203200 model_utils.py:486] epoch: 0  step: 427 of 427 metrics: eval/loss= 0.2500598728656769 eval/per_example_accuracy= 0.7901339530944824 eval/per_batch_alignment_identity= 0.9944719076156616 eval/yield_over_ccs= 3.6716418266296387
I1026 10:54:41.021777 140202426203200 model_utils.py:486] epoch: 1  step: 427 of 427 metrics: eval/loss= 0.24209719896316528 eval/per_example_accuracy= 0.8003109693527222 eval/per_batch_alignment_identity= 0.9945275187492371 eval/yield_over_ccs= 4.522387981414795
I1026 11:55:29.371924 140202426203200 model_utils.py:486] epoch: 1  step: 427 of 427 metrics: eval/loss= 0.20948448777198792 eval/per_example_accuracy= 0.8074153661727905 eval/per_batch_alignment_identity= 0.9950658082962036 eval/yield_over_ccs= 5.410595893859863
I1026 12:56:17.871346 140202426203200 model_utils.py:486] epoch: 1  step: 427 of 427 metrics: eval/loss= 0.23493772745132446 eval/per_example_accuracy= 0.8051523566246033 eval/per_batch_alignment_identity= 0.994662880897522 eval/yield_over_ccs= 5.366013050079346
I1026 13:57:06.771188 140202426203200 model_utils.py:486] epoch: 1  step: 427 of 427 metrics: eval/loss= 0.2536310851573944 eval/per_example_accuracy= 0.8128090500831604 eval/per_batch_alignment_identity= 0.9943390488624573 eval/yield_over_ccs= 5.1049723625183105
```

We mainly track eval/per_example_accuracy to assess the model's performance.
This metric represents the proportion of examples in the evaluation set that
have a 100% correct sequence.

You can also check roughly how long it is to run each epoch:

```
$ grep 'Starting to run epoch' /tmp/dc-tpu.log
I1026 05:48:53.098124 140202426203200 model_train_custom_loop.py:204] Starting to run epoch: 0
I1026 09:57:48.051159 140202426203200 model_train_custom_loop.py:204] Starting to run epoch: 1
I1026 14:04:58.815681 140202426203200 model_train_custom_loop.py:204] Starting to run epoch: 2
```

Based on this log, it takes a little longer than 4 hours to run one epoch (in
our setting here, an "epoch" means 100,000,000 examples).

## IMPORTANT: Remember to clean up the Cloud TPU VM and the persistent disk!

To check any Cloud TPU VMs that might be lying around:

```bash
gcloud compute tpus tpu-vm list \
  --zone ${ZONE} --project ${PROJECT}
```

To delete the Cloud TPU VM:

```bash
gcloud compute tpus tpu-vm delete ${USER}-tpu-name \
  --zone ${ZONE} --project ${PROJECT}
```

To delete the Cloud VM used for persistent disk setup:

```bash
gcloud compute instances delete ${USER}-disk-setup \
  --zone ${ZONE} --project ${PROJECT}
```

You can also use `stop` instead `delete` to stop the VM.

And, the disk can be cleaned up with:

```bash
gcloud compute disks delete ${USER}-tpu-disk \
  --zone ${ZONE} --project ${PROJECT}
```
