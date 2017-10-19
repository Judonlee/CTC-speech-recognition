# CTC-speech-recognition
This is a working example of using CTC for phone recognition on TIMIT.
CTC (connectionist temporal classification) is a sequence-to-sequence classifier, which maps an input sequence to a target sequence. In speech recognition, it predicts a sequence of labels (can be phones, or characters) from speech frames. The network typically uses bidirectional LSTM. For a detailed description of CTC, please look at this paper:
http://www.cs.toronto.edu/~graves/icml_2006.pdf
CTC eliminates the need for a seed HMM-GMM as in convensional ASR, making it much more concise. Also, if we use characters as the acoustic units, CTC also avoids the need of a pronunciation dictionary. In fact, CTC does not even need a language model, but having a language model improves recognition rate.

This example creates a CTC system for TIMIT phone recognition. it can be easily modified to do large vocabulary word recognition, by simply using character labels instead of phone labels. This paper http://proceedings.mlr.press/v32/graves14.pdf provides a detailed illustration of CTC-based word recognition.

Requirements:
1. This example uses pytorch and several other libaries. I recommend installing Anaconda 4.0.0 from https://repo.continuum.io/archive/ for Python 3. This saves you a lot of effort installing necessary packages yourself. The current Anaconda version is 4.4, but it seems the newest version has a problem when installing the CTC decoder later. Also, python 2.7 has a problem with the CTC decoder, which has not been clearly solved. So, to save effort, Anaconda 4.0.0 for Python 3.5 is the best starting point.
2. The code uses Pytorch. An easy installation can be found here http://pytorch.org/. You need to have GPU and CUDA.
3. Pytorch does not provide the CTC loss function. However, deepspeech has a pytorch wrapper for the CTC loss function. Follow the instructions at https://github.com/SeanNaren/deepspeech.pytorch to install the Warp-CTC module. After installation, copy libwarpctc.so to your python lib anaconda3/lib/python3.5/site-packages/
4. If you want to use the CTC beam decoder as in the paper http://proceedings.mlr.press/v32/graves14.pdf, you need to have the decoder libary installed. Follow the instructions at https://github.com/ryanleary/pytorch-ctc. However, if you want to use a language model, the decoder only accepts kenlm, rather than the ARPA format. In the third-party/kenlm directory of the above link, run compile_query_only.sh to compile the tool which converts an ARPA LM to the kenlm format. The compiled tool build_binary will be stored in third_parth/kenlm/bin/ .

Run the example:
1. I assume that you have TIMIT database. The first step is to generate 40-dimension log-mel (FBANK) speech features. The features should be saved in HTK format. This step is not included in the code, but can be easily done. After you have generated the HTK features, run preprocess_ctc.py. This script also requires the TIMIT labels in MLF format, which I have provided in ref61.mlf. Usually, we use 61 phones at training time. Also, I have provided the lists for training, validation and test data in train_fea.scp, cv_fea.scp and core_fea.scp. When training a neural network, we need to normalize the speech features by its global mean and variance. The mean and variance are pre-computed and saved in the file stat. The file stat is generated by HTK, using the command HCompV -A -T 1 -c output_dir -k *.%%% -q mv -S train_fea.scp. the mask *.%%% means the mean and variance are computed from all files having the same last 3 characters (which is fea) in this example. The output file name is default to fea. You can rename it to stat by mv output_dir/fea stat. The script preprocess_ctc.py does mean and variance normalization, and groups speech features with their target labels together into h5py chunks. Each chunk contains 200 randomly chosen utterances. The file hmmlist contains a list of all phones. The saved labels start with an index of 1, following the order specified in hmmlist. The label 0 is saved for the blank label in CTC training. If you want to skip this step, I have provided the processed features in train_ctc and cv_ctc.

2. To train the CTC model, run the script train_ctc.py. You can specify the LSTM layers, number of directions, etc at the beginning of this script. I assume that your features are 40-dimension. If not, you can change it in __main__. Note, the num_classes should be set to 61, not 62, because the network will automatically add 1 for the blank label. The details of the model is in set_model_ctc.py. This model follows a similar manner of deep speech. But it does not use the convolutional layer at front, and also, it operates on packed padded sequence to account for different lengths of sequences, which requires a lot less GPU memory than deep speech. The models will be saved in weights_ctc, in which I have generated some models already. After each epoch, we need to cross validate to prevent overfitting. The objective function for cross validation is the edit distance between the decoded phone sequence and the target phone sequence. We simply uses the greedy decoder (the phone with maximum probability at each frame, and removes duplicates and blank labels) for a quick decoding to predict the phone sequence in cross validation.


3. At test time, please run test_ctc.py. Three decoders are available: greedy decoder, beam search with and without a language model. The decoder classes are in ctc_decode.py. If you want to use a language model, the LM must be in kenlm format. You can easily convert the ARPA LM bigram_original by bin/build_binary bigram_original bigram.ken. To use a language model, we also need to have a trie file. However, it seems to generate this file, it only accepts characters as building blocks for words (the labels must be a string of single characters). So we have to map the TIMIT phones to some symbols. I have done that for you, and the mapped ARPA LM is bigram_symbol, and the map list is map_list. The other two decoders does not need this extra processing. The output MLF file is result.mlf, and you can run get_acc.sh to get the recognition accuracy. The script get_acc.sh first merge 61 phones to 39 phones by HLed, an HTK tool, and computes accuracy by HResult.


Some bench marks:
using the best model I provide, the phone accuracies on the core test set are: 79.04% (greedy decoder), 79.5% (both beam with/without lm).


4. The code can be easily modified to use characters as acoustic units. When you have word transcriptions, simply extracts and index the characters one by one, and append an extra label for the space after each word. 
