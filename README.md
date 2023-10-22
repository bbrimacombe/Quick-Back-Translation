# QBT: Quick Back-Translation for Unsupervised Machine Translation

![alt text](https://github.com/bbrimacombe/Quick-Back-Translation/blob/master/figs/qbt.jpg)

# Overview
Quick back-translation makes a simple change to the Tranformer architecture during unsupervised machine translation training. The encoder, a non-autoregressive model, is repurposed as a geneative model to supplement back-translation. This enhancement requires no changes to the model architecture, and any encoder-decoder Transformer can be trained with QBT. The present codebase is a simple implementation of QBT inside a forked [MASS](https://github.com/ishwnews/MASS) repository.

### Dependencies
Currently QBT is implemented for unsupervised NMT based on the codebase of [XLM](https://github.com/facebookresearch/XLM), and subsequently [MASS](https://github.com/ishwnews/MASS). The depencies are as follows:
- Python 3
- NumPy
- PyTorch (version 0.4 and 1.0)
- fastBPE (for BPE codes)
- Moses (for tokenization)
- Apex (for fp16 training)

### Data

We use the same BPE codes and vocabulary as XLM and MASS. To download the datasets the commands can be found in the XLM or MASS codebases. For example in English and French:

```
cd MASS-QBT

wget https://dl.fbaipublicfiles.com/XLM/codes_enfr
wget https://dl.fbaipublicfiles.com/XLM/vocab_enfr

./get-data-nmt.sh --src en --tgt fr --reload_codes codes_enfr --reload_vocab vocab_enfr
```
### Overview
In order to implemented QBT into the original XLM or MASS codebases, these files have been updated:
- /train.py
- /src/trainer.py
- /src/model/transformer.py
- /src/data/loader.py
- /src/evaluation/evaluator.py  

### Encoder Initialization

If using QBT on a pre-trained language model which has never undergone UMT training, we must use some kind of initialization such that the encoder generates loosely translated-outputs. An approach proposed in the QBT manuscript is to warm up the encoder very briefly by mapping from randomly paired monolingual sequences of two seperate languages. This serves to align the modes of the languages: for example, "le" and "the" will appear in similar frequencies for English in French. Below find a command which serves as an effective warmup. Remember to replace the model with your checkpoint of interest.

```
python train.py --exp_name unsupMT_ende --data_path ./data/processed/de-en/ --lgs en-de --encoder_only false --emb_dim 1024 --n_layers 6 --n_heads 8 --dropout 0.1 --attention_dropout 0.1 --gelu_activation true --tokens_per_batch 2000 --batch_size 32 --bptt 256 --optimizer adam_inverse_sqrt,beta1=0.9,beta2=0.98,lr=0.0001 --epoch_size 5000 --max_epoch 200000 --eval_bleu true --reload_model .\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth,.\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth --encoder_mt_random_steps en-de,de-en
```

### QBT: Encoder Back-Translation

Using a model which has undergone some UMT training or encoder initialization, we may proceed to train the encoder as a super-fast generative model.

```
python train.py --exp_name unsupMT_ende --data_path ./data/processed/de-en/ --lgs en-de --encoder_only false --emb_dim 1024 --n_layers 6 --n_heads 8 --dropout 0.1 --attention_dropout 0.1 --gelu_activation true --tokens_per_batch 2000 --batch_size 32 --bptt 256 --optimizer adam_inverse_sqrt,beta1=0.9,beta2=0.98,lr=0.0001 --epoch_size 100000 --max_epoch 200000 --eval_bleu true --reload_model .\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth,.\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth  --encoder_bt_steps en-de-en,de-en-de
```

### QBT: Encoder Back-Translated Distillation

And finally, to use the encoder's fast generations to train the decoder, please see the following command:
```
python train.py --exp_name unsupMT_ende --data_path ./data/processed/de-en/ --lgs en-de --encoder_only false --emb_dim 1024 --n_layers 6 --n_heads 8 --dropout 0.1 --attention_dropout 0.1 --gelu_activation true --tokens_per_batch 2000 --batch_size 32 --bptt 256 --optimizer adam_inverse_sqrt,beta1=0.9,beta2=0.98,lr=0.0001 --epoch_size 1000 --max_epoch 200000 --eval_bleu true --reload_model .\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth,.\\dumped\\unsupMT_ende\\0aod9wqev7\\checkpoint.pth  --qbt_steps en-de-en,de-en-de
```

### QBT-Staged
In order to fully exploit the advantages of QBT, these optimizations are used in succession. QBT-Stages first initializes the encoder, trains with EBT, and then runs QBT and BT simultaneously, with the BT gradients to the encoder frozen. The final QBT command will include:

```
--qbt_steps en-de-en,de-en-de --bt_steps en-de-en,de-en-de
```
Please note that the gradients to the encoder during BT may be easily toggled inside the "bt_step".
After QBT-Stages, BT gradients may be turned on again, and BT may be run for final fine-tuning.

### QBT-Sync
Seperately from QBT-Staged, QBT-Sync proposes to use the QBT methods on a fully converged UMT model. Instead of freezing the BT gradients, all gradients are allowed to flow through the encoder.
```
--qbt_steps en-de-en,de-en-de --bt_steps en-de-en,de-en-de --encoder_bt_steps en-de-en,de-en-de
```

#### Distributed Training

Please refer to the MASS documentation for how to use distributed training in this codebase.

## Reference

If you find QBT useful in your research, you may cite the paper as:

... (TBD)
    


