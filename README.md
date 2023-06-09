# Project Descripton

This project includes three different subprojects:

-   A frequent subgraph mining project based on [gSpan](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=1184038&casa_token=hII_fVgAeycAAAAA:bc095z3FKW0KFzRYjy1yIcNp7jIKEy9jXd_9cv4FxosnXtFOkIHTkcy0cS8VJqrUK52z7aHIzw&tag=1)
-   A discriminative subgraph mining project based on [GAIA](https://dl.acm.org/doi/pdf/10.1145/1807167.1807262?casa_token=G-mvu1_NCJgAAAAA:4ToovRUGv-i_YrmH8SxNQP15J3hTY-N0rq49oNgp1h45khNM6c3cgwcQjKGe66q-05jZeFXg8Kfr)
-   A `pytorch` environment that encodes features (preprocesses graph data) produced by `gSpan` and `GAIA` and trains on processed data

# Setup

-   gSpan setup
    -   Create a virtual environment `env` within `gSpan` project by running `python3 -m venv env`.
    -   Install packages by running `pip install -r requirements.txt`
    -   Replace the files `gspan.py` and `graph.py` at `env/lib/python3/site-packages/gspan_mining/` with the files provided at `gspan/nx_support/`
-   GAIA setup
    -   Download link and instruction of GAIA can be found [here](https://sourceforge.net/projects/discriminatives/)
-   PyTorch Environment
    -   Create a virtual environment `env` within `pytorch` project by running `python3 -m venv env`.
    -   Install packages by running `pip install -r requirements.txt`

# Experiments

## 1. Generating Relational Features

### Preparing data for gSpan

Since we use TUD Benchmark, we need to convert TUD data to [Networkx](https://networkx.org/documentation/stable/index.html) data. To do so run the following command within the `pytorch` project.

```bash
# python utilities/convert.py -d {dataset_name}
# for example if we want to convert MUTAG to Networkx, run the following
python utilities/convert.py -d MUTAG
```

This command will convert TUD to Networkx and it will save a `{dataset}.dat` file under `pytorch/data_networkx/{dataset}.dat`.

Next, copy the generated `{dataset}.dat` file to `gspan/data_nx/{dataset}/{dataset}.dat`. Well-known dataset `MUTAG` already exists as an example.

Next we have to convert `networkx` data to a specific format that can be used with `gSpan`. To do so run the following command within the `gSpan` project.

```bash
# python utilities/nx_to_graph.py -d {dataset_name}
# for example if we want to convert MUTAG to Networkx, run the following
python utilities/nx_to_graph.py -d MUTAG
```

This will produce a `{dataset}.graph` file under `gspan/data_graph/{dataset}.graph`.

### Mining Subgraphs with gSpan

Now that we have `.graph` data we can generate relational features by running the following command within the `gspan` project.

```bash
# python app.py -d {dataset_name} -s {support_in_percentage} -n {length_of_dataset/nr_of_graphs}
# for example if we want to generate subgraphs of MUTAG with 90%, run the following
python app.py -d MUTAG -s 90 -n 188
```

This command should generate 13 subgraphs and save them under `/data_nx_features/{dataset_support}.dat`.

### Mining Discriminative Subgraphs with GAIA

To generate files that are used as an input for `GAIA` software, run the following command withing the `pytorch` environment. This command with automatically convert `TUD` datasets to a specific format used in `GAIA`. Two files will be generated `{dataset_name}_node_file.txt` and `{dataset_name}_edge_file.txt`.

```bash
# python utilities/tud_to_gaia.py -d {dataset_name}
# for example if we want to convert MUTAG to GAIA, run the following
python utilities/tud_to_gaia.py -d MUTAG
```

Once the files are ready, move them to `GAIA` directory and follow the instructions on how to use `GAIA`.

For `MUTAG` dataset, `GAIA` should generate six discriminative features. Generated features are saved in a file under the name `patternResult.txt`. Now, to convert these features to `Networkx` format move the file under `gspan/data_gaia/{dataset_name}/patternResult.txt` and run the following command within the `gspan` environment:

```bash
# python gaia_to_nx.py -d {dataset_name} -nf {nr_of_node_features/node_labels}
# for example if for MUTAG, we have 7 node features
python gaia_to_nx.py -d MUTAG -nf 7
```

This command will create a new file under `gspan/data_gaia/{dataset_name}/{dataset_name}_gaia_nx.dat`.

### Subgraph and Feature Selection

To make use of feature selection methods, we need to _flatten_ the graphs and their respective features genereated by one of the methods listed above.

Feature Selection make use of target variable of the dataset, so we need to extract that first. To do so, run the following command within the `pytorch` environment:

```bash
# python utilities/target_extraction.py -d {dataset_name}
# for example utilities/target_extraction.py -d MUTAG
python utilities/target_extraction.py -d {dataset_name}
```

This command will extract the target value and save it in a file under `data_networkx/{dataset_name}_target.dat`. Now the feature selection methods exist within the `gspan` environment. To make use of the feature selection method, first create a `playground`, for example: `gspan/mutag_playground`. In this directory, add the following files: `MUTAG.dat` which can be found under the `gspan/data_nx/MUTAG/MUTAG.dat` and the generated file that contains the target of the dataset at the `pytorch/data_networkx/MUTAG_target.dat`. The other file needed is the set of subgraphs/features. For example, if you set the support of `gSpan` to `20` for `MUTAG` dataset, it will produce `39492` subgraphs. In principle, any set of subgraphs can be used with feature selection techniques, but usually is used when the set of subgraphs is very large. Once you have the following structure in one of the playgrounds:

```
gspan/
  mutag_playground/
    MUTAG.dat # this is the set of graphs
    MUTAG_target.dat # this is the target variable for each graph
    MUTAG_20.dat # this is the set of subgraphs generated by gSpan with support 20
```

simply run the following command:

```bash
# python {dataset_name}_playground/selection.py -d {dataset_name} -f {set_of_features} -s {selection_method} -k {top_k_features}
# for example mutag1_playground/selection.py -d MUTAG -f MUTAG_20 -s xgb -k 20
python {dataset_name}_playground/selection.py -d {dataset_name} -f {set_of_features} -s {selection_method} -k {top_k_features}
```

This command will generate two files with first being a tabular representation (dataframe) of flattened graphs named `{dataset_name}_{set_of_features}_flatten_df.dat` and the set of selected features `{dataset_name}_{set_of_features}_{selection_method}_{top_k_features}.dat`. The later file can than be used for preprocessing the graphs.

## 2. Preprocessing TUD datasets and Training

### Creating an experiment

First, under `pytorch/experiments/` create a folder following your dataset name, e.g. `pytorch/experiments/MUTAG`. Inside this folder add the following structure:

```
pytorch/
  experiments/
    MUTAG/
      data/
      features/
      results/
      config.json
```

The directories `data` and `results` will be automatically populated once the experiments are executed. Under the `features` add the file generated by `gSpan` that contains your features `{dataset_support}.dat`. For example:

```
pytorch/
  experiments/
    MUTAG/
      data/
      features/
        MUTAG_90.dat
      results/
      config.json
```

The same way add features generated by `GAIA`.

The `config.json` contains configuration about the model hyperparameters and data preprocessing. For example, if we want to encode relational features from the file `MUTAG_90.dat`, with and without labels, the following configuration should be provided.

```json
{
    "model": {
        "seed": 0,
        "batch_size": 32,
        "hidden_channel": 32,
        "learning_rate": 0.001,
        "layers": 4,
        "scheduler": true,
        "step_size": 50,
        "decay": 0.9,
        "dropout": 0.5,
        "folds": 10,
        "epochs": 351
    },
    "features": [
        {
            "name": "MUTAG_90",
            "labelled": false,
            "device": "cuda:0"
        },
        {
            "name": "MUTAG_90",
            "labelled": true,
            "device": "cuda:1"
        }
    ]
}
```

Once the `config.json` is ready, first preprocess the data by running the following command within `pytorch` project:

```bash
#python preprocess.py -d {experiment}
python preprocess.py -d MUTAG
```

This command will preprocess the data and provide the new processed dataset under the `pytorch/experiments/{dataset}/data/processed/`

Now to train, simply run the following command within `pytorch` project:

```bash
#python preprocess.py -d {experiment}
python train.py -d MUTAG
```

This command will train based on the configurations given in the `config.json` file of an experiment. This will generate results based on the dataset name under the `pytorch/experiments/{dataset}/results/{dataset}_{label}.json`.
