Code for our EMNLP 2022 paper "Towards Robust k-Nearest-Neighbor Machine Translation".


The source code is developed upon adaptive kNN-MT.
You can see detail in https://github.com/zhengxxn/adaptive-knn-mt.

## Requirements and Installation

* pytorch version >= 1.5.0
* python version >= 3.6
* faiss-gpu >= 1.6.5
* pytorch_scatter = 2.0.5
* 1.19.0 <= numpy < 1.20.0

You can install this project by
```
pip install --editable ./
```

## Instructions

We use an example to show how to use our codes.

### Pre-trained Model and Data

The pre-trained translation model can be downloaded from [this site](https://github.com/pytorch/fairseq/blob/master/examples/wmt19/README.md).
We use the De->En Single Model for all experiments.

For convenience, the pre-processed data can be downloaded from [this site](https://drive.google.com/file/d/18TXCWzoKuxWKHAaCRgddd6Ub64klrVhV/view?usp=sharing).

### Create Datastore

This script will create datastore (includes key.npy and val.npy) for the data.

```
DSTORE_SIZE=3613350
MODEL_PATH=/path/to/pretrained_model_path
DATA_PATH=/path/to/fairseq_preprocessed_data_path
DATASTORE_PATH=/path/to/save_datastore
PROJECT_PATH=/path/to/ada_knnmt

mkdir -p $DATASTORE_PATH

CUDA_VISIBLE_DEVICES=0 python $PROJECT_PATH/save_datastore.py $DATA_PATH \
    --dataset-impl mmap \
    --task translation \
    --valid-subset train \
    --path $MODEL_PATH \
    --max-tokens 4096 --skip-invalid-size-inputs-valid-test \
    --decoder-embed-dim 1024 --dstore-fp16 --dstore-size $DSTORE_SIZE --dstore-mmap $DATASTORE_PATH
 
# 4096 and 1024 depend on your device and model separately
```

The DSTORE_SIZE depends on the num of tokens of target language train data. You can get it by two ways:

- find it in preprocess.log file, which is created by fairseq-process and in data binary folder.
- calculate wc -l + wc -w of raw data file.

The datastore sizes we used in our paper are listed as below:

| IT      | Medical | koran  | Law      |
|---------|---------|--------|----------|
| 3613350 | 6903320 | 524400 | 19070000 |

### Build Faiss Index

This script will build faiss index for keys, which is used for fast knn search. when the knn_index is build, you can
remove keys.npy to save the hard disk space.

```
PROJECT_PATH=/path/to/ada_knnmt
DSTORE_PATH=/path/to/saved_datastore
DSTORE_SIZE=3613350

CUDA_VISIBLE_DEVICES=0 python $PROJECT_PATH/train_datastore_gpu.py \
  --dstore_mmap $DSTORE_PATH \
  --dstore_size $DSTORE_SIZE \
  --dstore_fp16 \
  --faiss_index ${DSTORE_PATH}/knn_index \
  --ncentroids 4096 \
  --probe 32 \
  --dimension 1024
```

### Train Our Model
```
DSTORE_SIZE=3613350
DATA_PATH=/path/to/fairseq_preprocessed_data_path
PROJECT_PATH=/path/to/ada_knnmt
MODEL_PATH=/path/to/pretrained_model_path
DATASTORE_PATH=/path/to/saved_datastore

max_k_grid=(4 8 16 32)
batch_size_grid=(32 32 32 32)
update_freq_grid=(1 1 1 1)
valid_batch_size_grid=(32 32 32 32)

for idx in ${!max_k_grid[*]}
do

  MODEL_RECORD_PATH=/path/to/save/model/train-hid32-maxk${max_k_grid[$idx]}
  TRAINING_RECORD_PATH=/path/to/save/tensorboard/train-hid32-maxk${max_k_grid[$idx]}
  mkdir -p "$TRAINING_RECORD_PATH"

  CUDA_VISIBLE_DEVICES=0 python \
  $PROJECT_PATH/fairseq_cli/train.py \
  $DATA_PATH \
  --log-interval 100 --log-format simple \
  --arch transformer_wmt19_de_en_with_datastore \
  --tensorboard-logdir "$TRAINING_RECORD_PATH" \
  --save-dir "$MODEL_RECORD_PATH" --restore-file "$MODEL_PATH" \
  --reset-dataloader --reset-lr-scheduler --reset-meters --reset-optimizer \
  --validate-interval-updates 100 --save-interval-updates 100 --keep-interval-updates 1 --max-update 5000 --validate-after-updates 1000 \
  --save-interval 10000 --validate-interval 100 \
  --keep-best-checkpoints 1 --no-epoch-checkpoints --no-last-checkpoints --no-save-optimizer-state \
  --train-subset valid --valid-subset valid --source-lang de --target-lang en \
  --criterion label_smoothed_cross_entropy --label-smoothing 0.001 \
  --max-source-positions 1024 --max-target-positions 1024 \
  --batch-size "${batch_size_grid[$idx]}" --update-freq "${update_freq_grid[$idx]}" --batch-size-valid "${valid_batch_size_grid[$idx]}" \
  --task translation \
  --optimizer adam --adam-betas "(0.9, 0.98)" --adam-eps 1e-08 --min-lr 3e-05 --lr 0.0003 --clip-norm 1.0 \
  --lr-scheduler reduce_lr_on_plateau --lr-patience 5 --lr-shrink 0.5 \
  --patience 30 --max-epoch 500 \
  --load-knn-datastore --dstore-filename $DATASTORE_PATH --use-knn-datastore \
  --dstore-fp16 --dstore-size $DSTORE_SIZE --probe 32 \
  --knn-sim-func do_not_recomp_l2 \
  --use-gpu-to-search --move-dstore-to-mem  \
  --knn-lambda-type trainable --knn-temperature-type fix --knn-temperature-value 10 --only-train-knn-parameter \
  --knn-k-type trainable --k-lambda-net-hid-size 32 --k-lambda-net-dropout-rate 0.0 --max-k "${max_k_grid[$idx]}" --k "${max_k_grid[$idx]}" \
  --label-count-as-feature
done
```

### Inference with Our Model

```
DSTORE_SIZE=3613350
MODEL_PATH=/path/to/trained_model

DATASTORE_PATH=/path/to/datastore
DATA_PATH=/path/to/data
PROJECT_PATH=/path/to/ada_knnmt

OUTPUT_PATH=/path/to/save_output_result

mkdir -p "$OUTPUT_PATH"

CUDA_VISIBLE_DEVICES=0 python $PROJECT_PATH/experimental_generate.py $DATA_PATH \
    --gen-subset test\
    --path "$MODEL_PATH" --arch transformer_wmt19_de_en_with_datastore \
    --beam 4 --lenpen 0.6 --max-len-a 1.2 --max-len-b 10 --source-lang de --target-lang en \
    --scoring sacrebleu \
    --batch-size 32 \
    --tokenizer moses --remove-bpe \
    --model-overrides "{'load_knn_datastore': True, 'use_knn_datastore': True,
    'dstore_filename': '$DATASTORE_PATH', 'dstore_size': $DSTORE_SIZE, 'dstore_fp16': True, 'probe': 32,
    'knn_sim_func': 'do_not_recomp_l2', 'use_gpu_to_search': True, 'move_dstore_to_mem': True, 'no_load_keys': False,
    'knn_temperature_type': 'fix', 'knn_temperature_value': 10,}" \
    | tee "$OUTPUT_PATH"/generate.txt
```