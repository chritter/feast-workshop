# Module 0: Setting up a feature repo, sharing features, batch predictions

Welcome! Here we use a basic example to explain key concepts and user flows in Feast. 

We focus on a specific example (that does not include online features + models):
- **Use case**: building a platform for data scientists to share features for training offline models
- **Stack**: you have data in a combination of data warehouses (to be explored in a future module) and data lakes (e.g. S3)

# Mapping to Feast concepts
To support this, you'll need:
| Concept         | Requirements                                                                                                                                                                                     |
| :-------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Data sources    | `FileSource` (with S3 paths and endpoint overrides) and `FeatureView`s registered with `feast apply`                                                                                             |
| Feature views   | Feature views tied to data sources that are shared by data scientists, registered with `feast apply`                                                                                             |
| Provider        | In `feature_store.yaml`, specifying the `aws` provider to ensure your registry can be stored in S3                                                                                               |
| Registry        | In `feature_store.yaml`, specifying a path (within an existing S3 bucket) the registry is written to. Users + model servers will pull from this to get the latest registered features + metadata |
| Transformations | Feast supports last mile transformations with `OnDemandFeatureView`s that can be re-used                                                                                                         |

# User flows
There are three user groups here worth considering. The ML platform team, the data scientists, and the ML engineers scheduling models in batch. 

## User flow 1: ML Platform Team
The team here sets up the centralized Feast feature repository in GitHub. This is what's seen in `feature_repo_aws/`.

### Step 0: Setup S3 bucket for registry and file sources
This assumes you have an AWS account & Terraform setup. If you don't:
- Set up an AWS account & setup your credentials as per the [AWS quickstart](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)
- Install [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli#install-terraform)


We've made a simple Terraform project to help with S3 bucket creation:
```bash
cd infra/
terraform init
terraform apply
```

Output:
```bash
aws_s3_bucket.feast_bucket: Creating...
aws_s3_bucket.feast_bucket: Creation complete after 3s [id=feast-workshop-danny]
aws_s3_bucket_acl.feast_bucket_acl: Creating...
aws_s3_object.driver_stats_upload: Creating...
aws_s3_bucket_acl.feast_bucket_acl: Creation complete after 0s [id=feast-workshop-danny,private]
aws_s3_object.driver_stats_upload: Creation complete after 1s [id=driver_stats.parquet]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

project_bucket = "s3://feast-workshop-danny"
project_name = "danny"
```

### Step 1: Setup the feature repo
The first thing a platform team needs to do is setup the `feature_store.yaml` within a version controlled repo like GitHub. We've setup a sample feature repository in `feature_repo_aws/`

There are two files in `feature_repo_aws` you need to change to point to your S3 bucket:

**data_sources.py**
```python
driver_stats = FileSource(
    name="driver_stats_source",
    path="s3://[INSERT YOUR BUCKET]/driver_stats.parquet",
    s3_endpoint_override="http://s3.us-west-2.amazonaws.com",  # Needed since s3fs defaults to us-east-1
    timestamp_field="event_timestamp",
    created_timestamp_column="created",
    description="A table describing the stats of a driver based on hourly logs",
    owner="test2@gmail.com",
)
```

**feature_store.yaml**
```yaml
project: feast_demo_aws
provider: aws
registry: s3://[INSERT YOUR BUCKET]/registry.pb
online_store: null
offline_store:
  type: file
flags:
  alpha_features: true
  on_demand_transforms: true
```

A quick explanation of what's happening in this `feature_store.yaml`:

| Key             | What it does                                                                         | Example                                                                                               |
| :-------------- | :----------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| `project`       | Gives infrastructure isolation via namespacing (e.g. online stores + Feast objects). | any unique name (e.g. `feast_demo_aws`)                                                               |
| `provider`      | Defines registry location & sets defaults for offline / online stores                | `gcp` enables registries in GCS and sets BigQuery + Datastore as the default offline / online stores. |
| `registry`      | Defines the specific path for the registry (local, gcs, s3, etc)                     | `s3://[YOUR BUCKET]/registry.pb`                                                                      |
| `online_store`  | Configures online store (if needed)                                                  | `null`, `redis`, `dynamodb`, `datastore`, `postgres`, `hbase` (each have their own extra configs)     |
| `offline_store` | Configures offline store, which executes point in time joins                         | `bigquery`, `snowflake.offline`,  `redshift`, `spark`, `trino`  (each have their own extra configs)   |
| `flags`         | (legacy) Soon to be deprecated way to enable experimental functionality.             |                                                                                                       |

#### **Some further notes and gotchas**
- Generally, custom offline + online stores and providers are supported and can plug in. 
  - e.g. see [adding a new offline store](https://docs.feast.dev/how-to-guides/adding-a-new-offline-store), [adding a new online store](https://docs.feast.dev/how-to-guides/adding-support-for-a-new-online-store)
- **Project**
  - Users can only request features from a single project
- **Provider**
  - Default offline or online store choices can be easily overriden in `feature_store.yaml`. 
    - For example, one can use the `aws` provider and specify Snowflake as the offline store:
      ```yaml
      project: feast_demo_aws
      provider: aws
      registry: s3://[INSERT YOUR BUCKET]/registry.pb
      online_store: null
      offline_store:
          type: snowflake.offline
          account: SNOWFLAKE_DEPLOYMENT_URL
          user: SNOWFLAKE_USER
          password: SNOWFLAKE_PASSWORD
          role: SNOWFLAKE_ROLE
          warehouse: SNOWFLAKE_WAREHOUSE
          database: SNOWFLAKE_DATABASE

      ```
- **Offline Store** 
  - We recommend users use data warehouses or Spark as their offline store for performant training dataset generation. 
    - In this workshop, we use file sources for instructional purposes. This will directly read from files (local or remote) and use Dask to execute point-in-time joins. 
  - A project can only support one type of offline store (cannot mix Snowflake + file for example)
  - Each offline store has its own configurations which map to YAML. (e.g. see [BigQueryOfflineStoreConfig](https://rtd.feast.dev/en/master/index.html#feast.infra.offline_stores.bigquery.BigQueryOfflineStoreConfig)):
- **Online Store**
  - You only need this if you're powering real time models (e.g. inferring in response to user requests)
    - If you are precomputing predictions in batch (aka batch scoring), then the online store is optional. You should be using the offline store and running `feature_store.get_historical_features`. We touch on this later in this module.
  - Each online store has its own configurations which map to YAML. (e.g. [RedisOnlineStoreConfig](https://rtd.feast.dev/en/master/feast.infra.online_stores.html#feast.infra.online_stores.redis.RedisOnlineStoreConfig))
With the `feature_store.yaml` setup, you can now run `feast apply` to create & populate the registry. 

### Step 2: Adding the feature repo to version control & set up CI/CD

The feature repository is already created for you on GitHub.

<img src="ci_cd_flow.png" width=600 style="padding: 10px 0">

We setup CI/CD to automatically manage the registry. You'll want e.g. a GitHub workflow that
- on pull request, runs `feast plan` 
- on PR merge, runs `feast apply`.

#### **feast plan**
Go ahead and run `feast plan` in your directory. Sample output:
```bash
Created entity driver
Created feature view driver_hourly_stats
Created feature service model_v2

No changes to infrastructure
```

We recommend automatically running `feast plan` on incoming PRs to describe what changes will occur when the PR merges. 
- This is useful for helping PR reviewers understand the effects of a change.
- One example is whether a PR may change features that are already depended on in production by another model (e.g. `FeatureService`). 

An example GitHub workflow (See [feast_plan.yml](../.github/workflows/feast_plan.yml), which is setup in this workshop repo)

```yaml
name: Feast plan

on: [pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Python
        id: setup-python
        uses: actions/setup-python@v2
        with:
          python-version: "3.7"
          architecture: x64

      # Get the registry from the previous step and run `feast plan`
      - uses: actions/checkout@v2
      - name: Install feast
        run: pip install feast
      - name: Capture `feast plan` in a variable
        id: feast_plan
        run: |
          body=$(cd module_0/feature_repo_aws; feast plan)
          body="${body//'%'/'%25'}"
          body="${body//$'\n'/'%0A'}"
          body="${body//$'\r'/'%0D'}"
          echo "::set-output name=body::$body"

      # Post a comment on the PR with the results of `feast plan`
      - name: Create comment
        uses: peter-evans/create-or-update-comment@v1
        if: ${{ steps.feast_plan.outputs.body }}
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ${{ steps.feast_plan.outputs.body }}
```

![](feast_plan_CI.png)

#### **feast apply**
This will parse the feature, data source, and feature service definitions and publish them to the registry. It may also setup some tables in the online store to materialize batch features to. 

Sample output of `feast apply`:
```bash
Registered entity driver_id
Registered feature view driver_hourly_stats
Deploying infrastructure for driver_hourly_stats
```

### Step 3 (optional): Access control for the registry
We don't dive into this deeply, but you don't want to allow arbitrary users to clone the feature repository, change definitions and run `feast apply`. Thus, you should lock down your registry (e.g. with an S3 bucket policy) to only allow changes from your CI/CD user and perhaps some ML engineers.

### Step 4 (optional): Setup a Web UI endpoint
Feast comes with an experimental Web UI. Users can already spin this up locally with `feast ui`, but you may want to have a Web UI that is universally available. Here, you'd likely deploy a service that runs `feast ui` on top of a `feature_store.yaml`, with some configuration on how frequently the UI should be refreshing its registry.

![Feast UI](sample_web_ui.png)

### Other best practices
Many Feast users use `tags` on objects extensively. Some examples of how this may be used:
- To give more detailed documentation on a `FeatureView`
- To highlight what groups you need to join to gain access to certain feature views.
- To denote whether a feature service is in production or in staging.

Additionally, users will often want to have a dev/staging environment that's separate from production. In this case, once pattern that works is to have separate projects:

```bash
├── .github
│   └── workflows
│       ├── production.yml
│       └── staging.yml
│
├── staging
│   ├── driver_repo.py
│   └── feature_store.yaml
│
└── production
    ├── driver_repo.py
    └── feature_store.yaml
```
## 2b. ML engineers

Data scientists or ML engineers can use the defined `FeatureService` (corresponding to model versions) and schedule regular jobs that generate batch predictions (or regularly retrain).  

Feast right now requires timestamps in `get_historical_features`, so what you'll need to do is append an event timestamp of `now()`. e.g.

```python
# Get the latest feature values for unique entities
entity_df = pd.DataFrame.from_dict({"driver_id": [1001, 1002, 1003, 1004, 1005],})
entity_df["event_timestamp"] = pd.to_datetime('now')
training_df = store.get_historical_features(
    entity_df=entity_df, features=store.get_feature_service("model_v2"),
).to_df()

# Make batch predictions
predictions = model.predict(training_df)
```

### A note on scalability
You may note that the above example uses a `to_df()` method to load the training dataset into memory and may be wondering how this scales if you have very large datasets.

`get_historical_features` actually returns a `RetrievalJob` object that lazily executes the point-in-time join. The `RetrievalJob` class is extended by each offline store to allow flushing results to e.g. the data warehouse or data lakes. 

Let's look at an example with BigQuery as the offline store. 
```yaml
project: feast_demo_gcp
provider: gcp
registry: gs://[YOUR BUCKET]/registry.pb
offline_store:
  type: bigquery
  location: EU
flags:
  alpha_features: true
  on_demand_transforms: true
```

Retrieving the data with `get_historical_features` gives a `BigQueryRetrievalJob` object ([reference](https://rtd.feast.dev/en/master/index.html#feast.infra.offline_stores.bigquery.BigQueryRetrievalJob)) which exposes a `to_bigquery()` method. Thus, you can do:

```python
path = store.get_historical_features(
    entity_df=entity_df, features=store.get_feature_service("model_v2"),
).to_bigquery()

# Continue with distributed training or batch predictions from the BigQuery dataset.
```

## 2c. Data scientists
Data scientists will be using or authoring features in Feast. They can similarly handle larger datasets with methods like `RetrievalJob#to_bigquery()` as described above.

There are two ways they can use Feast:
- Use Feast primarily as a way of pulling production ready features. 
  - See the `client/` folder for an example of how users can pull features by only having a `feature_store.yaml` 
  - This is **not recommended** since data scientists cannot register feature services to indicate they depend on certain features in production. 
- **[Recommended]** Have a local copy of the feature repository (e.g. `git clone`) and author / iterate / re-use features. 
  - Data scientist can:
    1. iterate on features locally
    2. apply features to their own dev project with a local registry & experiment
    3. build feature services in preparation for production
    4. submit PRs to include features that should be used in production (including A/B experiments, or model training iterations)

Data scientists can also investigate other models and their dependent features / data sources / on demand transformations through the repository or through the Web UI (by running `feast ui`)

# Conclusion
As a result:
- You have file sources (possibly remote) and a remote registry (e.g. in S3)
- Data scientists are able to author + reuse features based on a centrally managed registry. 
- ML engineers are able to use these same features with a reference to the registry to regularly generating predictions on the latest timestamp.
- You have CI/CD setup to automatically update the registry + online store infrastructure when changes are merged into the version controlled feature repo. 