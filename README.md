# Introduction 
This repository contains the code for *Instance-wise Ordered Transformer (IOT)*, which is introduced in the ICLR2021 paper [IOT: Instance-wise Layer Reordering for Transformer Structures](https://openreview.net/forum?id=ipUPfYxWZvM).

If you find this work helpful in your research, please cite as:
```
@inproceedings{
zhu2021iot,
title={{\{}IOT{\}}: Instance-wise Layer Reordering for Transformer Structures},
author={Jinhua Zhu and Lijun Wu and Yingce Xia and Shufang Xie and Tao Qin and Wengang Zhou and Houqiang Li and Tie-Yan Liu},
booktitle={International Conference on Learning Representations},
year={2021},
url={https://openreview.net/forum?id=ipUPfYxWZvM}
}
```

# Requirements and Installation
* PyTorch version == 1.0.0
* Python version >= 3.5

To install IOT:
```shell script
git clone https://github.com/instance-wise-ordered-transformer/IOT
cd IOT
pip install --editable .
```

# Getting Started
Take IWSLT14 De-En translation as an example.

### Data Preprocessing

```shell script
cd examples/translation/
bash prepare-iwslt14.sh
cd ../..

TEXT=examples/translation/iwslt14.tokenized.de-en
python preprocess.py --source-lang de --target-lang en \
    --trainpref $TEXT/train --validpref $TEXT/valid --testpref $TEXT/test \
    --destdir data-bin/iwslt14.tokenized.de-en --joined-dictionary
```

### Training
Encoder order is set to be the default one without reordering (```ENCODER_MAX_ORDER=1```), since the paper finds that both reordering encoder and decoder is not good as reordering decoder only. 
```shell script
#!/bin/bash
export CUDA_VISIBLE_DEVICES=${1:-0}
nvidia-smi

ENCODER_MAX_ORDER=1
DECODER_MAX_ORDER=3
DECODER_ORDER="0 3 5"
DIVERSITY=0.1
GS_MAX=20
GS_MIN=2
GS_R=0
GS_UF=5000
KL=0.01
CLAMPVAL=0.05

DECODER_ORDER_NAME=`echo $DECODER_ORDER | sed 's/ //g'`
SAVE_DIR=checkpoints/dec_${DECODER_MAX_ORDER}_order_${DECODER_ORDER_NAME}_div_${DIVERSITY}_gsmax_${GS_MAX}_gsmin_${GS_MIN}_gsr_${GS_R}_gsuf_${GS_UF}_kl_${KL}_clampval_${CLAMPVAL}
mkdir -p ${SAVE_DIR}

python -u train.py data-bin/iwslt14.tokenized.de-en -a transformer_iwslt_de_en \
--optimizer adam --lr 0.0005 -s de -t en --label-smoothing 0.1 --dropout 0.3 --max-tokens 4000 \
--min-lr 1e-09 --lr-scheduler inverse_sqrt --weight-decay 0.0001 --criterion label_smoothed_cross_entropy \
--max-update 100000 --warmup-updates 4000 --warmup-init-lr 1e-07 --adam-betas '(0.9,0.98)' \
--save-dir $SAVE_DIR --share-all-embeddings  --gs-clamp --decoder-orders $DECODER_ORDER  \
--encoder-max-order $ENCODER_MAX_ORDER  --decoder-max-order $DECODER_MAX_ORDER  --diversity $DIVERSITY \
--gumbel-softmax-max $GS_MAX  --gumbel-softmax-min $GS_MIN --gumbel-softmax-tau-r $GS_R  --gumbel-softmax-update-freq $GS_UF \
--kl $KL --clamp-value $CLAMPVAL | tee -a ${SAVE_DIR}/train.log


```

### Evaluation
```shell script
#!/bin/bash
set -x
set -e

pip install -e . --user
export CUDA_VISIBLE_DEVICES=${1:-0}
nvidia-smi

ENCODER_MAX_ORDER=1
DECODER_MAX_ORDER=3
DECODER_ORDER="0 3 5"
DIVERSITY=0.1
GS_MAX=20
GS_MIN=2
GS_R=0
GS_UF=5000
KL=0.01
CLAMPVAL=0.05

DECODER_ORDER_NAME=`echo $DECODER_ORDER | sed 's/ //g'`
SAVE_DIR=checkpoints/dec_${DECODER_MAX_ORDER}_order_${DECODER_ORDER_NAME}_div_${DIVERSITY}_gsmax_${GS_MAX}_gsmin_${GS_MIN}_gsr_${GS_R}_gsuf_${GS_UF}_kl_${KL}_clampval_${CLAMPVAL}

python generate.py data-bin/iwslt14.tokenized.de-en \
  --path $SAVE_DIR/checkpint_best.pt \
  --batch-size 128 --beam 5 --remove-bpe --quiet --num-ckts $DECODER_MAX_ORDER 
```
