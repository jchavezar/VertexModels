# Vertex AI Custom Container Deployment

**Snippets for Google Vertex AI!**

---

You can:

1. Package Assembly.
2. Train in Vertex Training.
3. Deploy it in Vertex Endpoints [here]

## Data Type

- http request in json format ({'instances' : [1,2,3,4]}) [more info](https://cloud.google.com/vertex-ai/docs/predictions/online-predictions-custom-models#formatting-prediction-input)

## Authentication

Just [login](https://console.cloud.google.com/) to GCP.

## Set your variables:

```shell
BUCKET_NAME=something                                                              # Bucket id in the following format: gs://NAME
```

## Step 1: Package Assemly.

<details>
<summary>"Before you start trainig"</summary>
<br />
These are the steps to build the package that contains the files for training (no need to wrap it up and containarize it).

Layout:
    
    - PKG-INFO
    - setup.cfg
    - setup.py
    - trainer
      - __init__.py
      - task.py
    
</details>

### Assemly the package:

```ruby
if [ ! -d custom ]; then
   mkdir custom;
fi
```

```ruby
cat << EOF > custom/setup.cfg
[egg_info]

tag_build =
tag_date = 0
EOF
```

```python
cat << EOF > custom/setup.py
import setuptools

setuptools.setup(
    install_requires=[
        'tensorflow==2.3.0',
    ],
    packages=setuptools.find_packages())
EOF
```

```ruby
cat << EOF > custom/PKG-INFO
Metadata-Version: 1.0

Name: Miles per galon classification
Version: 0.0.0
Summary: Demostration training script
Home-page: www.google.com
Author: Google
Author-email: jesusc@google.com
License: Public
Description: Demo
Platform: Vertex
EOF
```

```ruby
if [ ! -d custom/trainer ]; then
   mkdir custom/trainer;
fi
```

```ruby
touch custom/trainer/__init__.py
```

### Create training code:

```Python
cat << EOF > trainer/train.py
import warnings
import pandas as pd
import tensorflow as tf

warnings.filterwarnings('ignore')

from tensorflow import keras
from tensorflow.keras import layers
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('--model-dir', dest='model_dir',
                    default=os.getenv('AIP_MODEL_DIR'), type=str, help='Model dir.')
args = parser.parse_args()

# Extraction process
dataset = pd.read_csv('https://storage.googleapis.com/jchavezar-public-datasets/auto-mpg.csv')
dataset.tail()

dataset.isna().sum()
dataset = dataset.dropna()
dataset['Origin'] = dataset['Origin'].map({1: 'USA', 2: 'Europe', 3: 'Japan'})
dataset = pd.get_dummies(dataset, prefix='', prefix_sep='')
dataset.tail()

train_dataset = dataset.sample(frac=0.8, random_state=0)
test_dataset = dataset.drop(train_dataset.index)

train_stats = train_dataset.describe()
train_stats.pop("MPG")
train_stats = train_stats.transpose()

train_labels = train_dataset.pop('MPG')
test_labels = test_dataset.pop('MPG')


def norm(x):
    return (x - train_stats['mean']) / train_stats['std']


normed_train_data = norm(train_dataset)
normed_test_data = norm(test_dataset)


def build_model():
    model_ai = keras.Sequential([
        layers.Dense(64, activation='relu', input_shape=[len(train_dataset.keys())]),
        layers.Dense(64, activation='relu'),
        layers.Dense(1)
    ])
    optimizer = tf.keras.optimizers.RMSprop(0.001)
    model_ai.compile(loss='mse',
                     optimizer=optimizer,
                     metrics=['mae', 'mse'])
    return model_ai


model = build_model()
model.summary()
EPOCHS = 1000

# The patience parameter is the amount of epochs to check for improvement
early_stop = keras.callbacks.EarlyStopping(monitor='val_loss', patience=10)
early_history = model.fit(normed_train_data, train_labels,
                          epochs=EPOCHS, validation_split=0.2,
                          callbacks=[early_stop])

# Export model and save to GCS
print(args.model_dir)
model.save(args.model_dir)
EOF
```

### Store training script on your Google Cloud Storage:

```ruby
rm -f customer.tar custom.tar.gz
tar cvf custom.tar custom
gzip custom.tar
gsutil cp custom.tar.gz $BUCKET_NAME/trainer_iris.tar.gz
```

## Step 2: Train in Vertex Training.