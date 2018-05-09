# DeepCurator

A convolutional neural network trained to recognize good\* electronic music. Using freely available data from **Discogs** _'the largest music database and marketplace in the world'_ a dataset was crafted of the best and worst rated items tagged as 'Electronic'

Items were scored by taking the have/want ratio, average rating, number of ratings, and recommended sale price, and combining them to a normalized score. The top 15,000 scoring items were labelled as 'good' and the bottom 15,000 as 'not-good' (for lack of a better term)

For each item the audio was extracted using the associated YouTube link provided from community submissions via Discogs. To provide an input to the network log-scaled mel spectrograms were generated for 30 second slices from each track

After training, the neural network can be provided with an _unseen_ clip of audio and assign it a score. The higher the score, the more confidence it has of the audio being categorized as 'good'

\*Good is subjective, since taste is subjective. The scoring method is not perfect since it partly relies on the popularity of a release to assign a category. Also, rarer items can have an increased price because of their scarcity, not their audio qualities, which inflates the score erroneously. The scoring metric could be viewed more of a 'what will be sought-after on Discogs' rather than 'what is objectively good'

## Installation

This project uses Pipenv to manage dependencies. To install all dependencies run `pipenv install`. To activate the virtual environment run `pipenv shell`

Data files are stored on S3, to re-create the results, or train your own model, an configuration file will be needed. Create `config.yml` in the root of the project and fill in the following values.

```yaml
s3:
  region: 'your-bucket-region'
  access_key: 'aws-access-key'
  secret: 'aws-access-secret'
  bucket: 'bucket-to-store-data'
```

### Training the model

The Discogs dataset used to generate the labels is provided in `dataset.csv`. In here you will find over 90,000 items each with a Discogs ID, Haves, Wants, Rating Average, Rating Count, Price, and YouTube ID. From this a set of labels can be created by running `python create_labels.py`. This step is optional, as a pre-generated labels file is provided. However, you may wish to run this step if you want to modify the scoring function and re-score the items.

To train the model on your own dataset, you will need to create your own `labels.csv` file. This takes the shape of a comma seperated list of YouTube ID, and category (1 or 0).

Once you have a set of a labels, run the `download_data.py` script. It's recommended to run this on an EC2 instance with a high number of cores and a high network bandwidth. Multiprocessing is used to automatically scale to the number of available cores. Downloading 30,000 YouTube videos and extracting the mp3 from each can take a couple of hours even on a high spec cloud instance.

Spectrograms are generated by running `process_audio.py`. Again, use a high spec EC2 instance. This will grab each audio file from your specified bucket and generate a number of 30 second slices of the spectrogram for each track.

Finally, to train the network run `train.py`.

### Making predictions

A small Flask server is provided so predictions against unseen audio can be made by posting to a REST endpoint. A trained model is already provided in the repo, so you don't have to go through the training process if so desired.

To start the server run `server.py` this will load the trained model, ready for incoming events. The predictor takes in an mp3 file, splits it into _n_ slices, where n is the length of the audio divided by 30 seconds, and creates an average rating using the sum of the individual ratings for each slice. You can try it in your terminal by running:

```
curl -X POST -F audio=@"audio.mp3" 'http://localhost:5000/predict'
```

A score between 0 and 100 will be returned, where anything over 50 is what the model deems as 'good'
