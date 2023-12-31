# ssri_model

##  I. Predicting Serotonin Reuptake Inhibitor Bioactivity

This phase involves data collection, preprocessing, and generating molecular fingerprints for predicting bioactivity.

## Setup and Dependencies

Before starting the project, make sure you have the necessary dependencies installed. These include `chembl_webresource_client`, `pandas`, and `padelpy`.

```bash
pip install chembl_webresource_client pandas padelpy
```

## Data Collection and Preprocessing

### Load the Dataset

The initial steps involve loading data from the ChEMBL database using the `chembl_webresource_client`. The target serotonin inhibitors are queried and stored in a Pandas DataFrame.

```python
import pandas as pd
from chembl_webresource_client.new_client import new_client

serotonin_target = new_client.target
serotonin_targets = serotonin_target.search('serotonin')
sert_mm_df = pd.DataFrame.from_dict(serotonin_targets)
```

### Data Cleaning

After loading the dataset, data cleaning steps are performed to prepare it for further analysis. This includes removing missing values and duplicate molecules.

```python
clean_standard_value = sert_mm_by_ki_df[sert_mm_by_ki_df.standard_value.notna()]
clean_canonical_smiles = clean_standard_value[clean_standard_value.canonical_smiles.notna()]
clean_repeated_molecules = clean_canonical_smiles.drop_duplicates(subset=['molecule_chembl_id'], keep='first')
```

### Export Clean Dataset

The cleaned dataset is exported to a CSV file for future use.

```python
clean_repeated_molecules.to_csv("clean_serotonin_target.csv", index=False)
```

## Generating Molecular Fingerprints

### Prepare Dataset for Descriptor Calculation

The dataset is prepared by concatenating the necessary columns and exporting them to a SMILES file for descriptor calculation.

```python
smiles_n_id = pd.concat([clean_repeated_molecules['canonical_smiles'], clean_repeated_molecules['molecule_chembl_id']], axis=1)
smiles_n_id.to_csv('molecule.smi', sep='\t', index=False, header=False)
```

### Generate KlekothaRothCount Fingerprint

The KlekothaRothCount molecular fingerprint descriptor is generated using the `padelpy` library. The XML descriptor file is obtained and used for generating the fingerprints.

```python
kkrc_xml = glob.glob("*.xml")

padeldescriptor(mol_dir='molecule.smi', 
                d_file='KlekotaRothFingerprintCountDescriptor.csv', 
                descriptortypes=kkrc_xml[0],
                detectaromaticity=True,
                standardizenitro=True,
                standardizetautomers=True,
                threads=2,
                removesalt=True,
                log=True,
                fingerprints=True
)
```

### Import Descriptor Data

The generated descriptors are imported from the CSV file.

```python
descriptors = pd.read_csv('KlekotaRothFingerprintCountDescriptor.csv')
```

## II. Data Preprocessing and Feature Engineering

This phase outlines the process of data preprocessing and feature engineering for a machine learning project involving the prediction of bioactivity (Ki) of serotonin reuptake inhibitors. The project includes steps such as data import, normalization and dataset concatenation.

## Setup and Dependencies

Ensure you have the necessary dependencies installed, including `pandas`, `numpy`, `matplotlib`, `seaborn`, and `sklearn`.

```bash
pip install pandas numpy matplotlib seaborn scikit-learn
```

## Data Import and Exploration

### Import Data

The initial step involves importing the necessary data using the Pandas library.

```python
import pandas as pd

# Import Klekota-Roth Fingerprint Descriptor Data
klekothar_roth = pd.read_csv('KlekotaRothFingerprintCountDecriptor.csv')

# Import Bioactivity (Ki) Data
ki = pd.read_csv('serotonin_clean_needed_dataset.csv')
```

### Data Exploration

After importing the data, you can explore its structure and contents using the Pandas functions.

```python
klekothar_roth.shape
ki.shape
```

## Data Preprocessing

### Normalize Ki Values

To normalize the bioactivity (Ki) values, the Min-Max Scaling technique is used.

```python
from sklearn.preprocessing import MinMaxScaler

# Initialize MinMaxScaler
scaler = MinMaxScaler()

# Fit and transform Ki values
new_y = pd.DataFrame(scaler.fit_transform(ki))
```

## Feature Engineering

### Prepare Feature and Target Datasets

The feature dataset (`X`) is created by dropping the 'Name' column from the Klekota-Roth Fingerprint Descriptor data.

```python
X = klekothar_roth.drop('Name', axis=1)
```

### Concatenate Feature and Target Datasets

The feature dataset (`X`) and the transformed bioactivity values (`y`) are concatenated to create a single dataset with both features and the target variable.

```python
descriptor_and_ki = pd.concat([X, y], axis=1)
```

### Export Processed Dataset

The processed dataset containing descriptors and normalized bioactivity values is exported to a CSV file.

```python
descriptor_and_ki.to_csv('descriptor_and_ki.csv', index=False)
```


##  III. MindsDB Model Training and Prediction

This documentation outlines the process of connecting to MindsDB, creating and training a model, and making predictions using the MindsDB SDK.

## Setup and Dependencies

Before starting the project, make sure you have the `mindsdb_sdk` library installed. You can install it using the following command:

```bash
pip install mindsdb_sdk
```

## Connecting to MindsDB

The initial steps involve connecting to the MindsDB server using your username and password.

```python
import mindsdb_sdk
from getpass import getpass

# Enter your MindsDB login credentials
username = 'adewarainioluwa@gmail.com'
password = getpass('Enter your MindsDB password: ')

# Connect to MindsDB
server = mindsdb_sdk.connect(login=username, password=password)
```

## Model Creation and Training

### Get Project and Table

After connecting to MindsDB, retrieve the project and table data.

```python
project = server.get_project('ssri_drug_discovery_project')
files_db = server.get_database('files')
ssri_table = files_db.get_table('fingerprint_descriptor_and_ki')
```

### Check for Existing Model

Check if the desired model already exists. If not, create the model.

```python
model_names = [i.name for i in project.list_models()]

if 'ssri_model' in model_names:
    print('ssri_model already exists. Returning existing model')
    ssri_model = project.get_model('ssri_model')
else:
    print('Creating ssri_model model')
    ssri_model = project.create_model(
        name='ssri_model',
        predict='0',
        query=ssri_table
    )
```

### Wait for Model Training

Wait for the model to finish training and check for any errors.

```python
print('Waiting for model training to complete...please be patient')
for i in range(400):
    print('.', end='')
    time.sleep(0.5)

    if ssri_model.get_status() not in ('generating', 'training'):
        print('\nFinished training ssri_model:')
        break

if ssri_model.get_status() == 'error':
    print('Something went wrong while training:')
    print(ssri_model.data['error'])
```

## Making Predictions

### Fetch Data and Prepare for Prediction

Fetch data from the table and prepare it for making predictions.

```python
df = ssri_table.fetch()
df = df[:3]
```

### Predict Using the Model

Make predictions using the trained model.

```python
result = ssri_model.predict(df)
```

### Export Predictions

Export the prediction results to a CSV file.

```python
result.to_csv("result.csv", index=False)
```
