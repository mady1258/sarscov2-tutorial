# Customizing analysis  

## Changing parameters  
Each step in the [augur workflow](orientation-workflow.md) can be parameterized; these parameters are specified in `config.yaml` files.  

We've provided reasonable default values for each step in the `default_config/default_config.yaml`; these are largely the same values we use for our [nextstrain.org/ncov workflows](https://github.com/nextstrain/ncov).

We encourage you to take a few minutes to **skim through [the default config file](default_config/default_config.yaml). Although these default values should be fine for most users, it's helpful to get a sense for what options are available.**  

If you'd like to tweak the parameterization, **you can override any of these values by specifying them in the `my_analyses/<name>/config.yaml` file. Any values not overridden in this way will fall back to the default values.**
Keeping build-specific parameters separate this way prevents mixups of settings between builds, and gives you a cleaner file to work with (rather than having to wrestle the _entire_ default config file).

## Adding a new place    

Places are defined as one of:  
- `region` (e.g., `North America`, `Asia`)  
- `country  
- `division` (i.e., state, province, or canton)  
- `location` (i.e., a county or city within a division)  

To define a new place, you'll need to specify its GPS coordinates and a color.  

1. Add a line to `default_config/lat_longs.tsv`. This file is separated into sections for each geographic resolution. This looks like:  
```
# resolution	place	latitude	longitude
location	Abondant	48.790785	1.420178
```

>Note: keep in mind that `0.0` longitude is the prime meridian; to specify something in the Western hemisphere, you'll need to enter a _negative_ value for longitude. Similarly, to specify something in the Southern hemisphere, you'll need to enter a _negative_ value for latitude

2. Add an entry to `ordering.tsv` such that your newly-defined place is next to geographically nearby places in the list.  


## Subsampling  

### Basic subsampling  
Reasonable defaults are pre-defined. You can find a [description of them here](XXX).

### Custom subsampling schemes
We implement hierarchical subsampling by producing multiple samples at different geographic scales
and merge these samples into one file for further analysis.
A build can specify any number of such samples which can be flexibly restricted to particular
meta data fields and subsampled from groups with particular properties.
When specifying subsampling in this way, we'll first take sequences from the 'focal' area, and the select samples from other geographical areas.
Read further for information on how we select these samples.

In this example, we'll look at a subsampling scheme which defines a `canton`.
Cantons are regional divisions in Switzerland - below 'country,' but above 'location' (often city-level).
Here, we'd like to be able to specify a particular 'canton' and do focal sampling there, with contextual samples from elsewhere in the country, other countries in the region, and other regions in the world.

For cantons this looks like this:
```yaml
subsampling:
  # We are calling this type of sampling 'canton'.
  # As a 'canton' is a division, we'll start by defining what kind of sampling we want at the 'division' level, for whatever canton we specify
  canton: ## Build name
    # Focal samples for division (only samples from a specifed division with 300 seqs per month)
    division:
      group_by: "year month"
      seq_per_group: 300
      exclude: "--exclude-where 'region!={{region}}' 'country!={{country}}' 'division!={{division}}'"
    # Now we'll specify the types of 'contextual' samples we want:
    # Contextual samples from division's country
    country:
      group_by: "division year month"
      seq_per_group: 20
      exclude: "--exclude-where 'region!={{region}}' 'country!={{country}}' 'division={{division}}'"
      priorities:
        type: "proximity"
        focus: "division"
    # Contextual samples from division's region
    region:
      group_by: "country year month"
      seq_per_group: 10
      exclude: "--exclude-where 'region!={{region}}' 'country={{country}}'"
      priorities:
        type: "proximity"
        focus: "division"
    # Contextual samples from the rest of the world, excluding the current
    # division to avoid resampling.
    global:
      group_by: "country year month"
      seq_per_group: 5
      exclude: "--exclude-where 'region={{region}}'"
      priorities:
        type: "proximity"
        focus: "division"
```
All entries above canton level (the 'contextual' samples) specify priorities. Currently, we have only implemented
one type of priority called `proximity`.
It attempts to selected sequences as close as possible to the focal samples
specified as `focus: division`.
The argument of the latter has to match the name of one of the other subsamples. 

If you need parameters in a way that isn't represented by the configuration file, [create a new issue in the ncov repository](https://github.com/nextstrain/ncov/issues/new) to let us know.


## Ancestral trait reconstruction  

Trait reconstruction is the process by which augur infers the most likely metadata value of an internal node. For example, if an internal node (which always represents a hypothesized, ancestral virus / case) has 3 descendants, all of which were isolated in Washington State, we might infer that the ancestor was most likely also circulating in Washington State (see ["Interpretation"](XXX) for more).

For each build, you can specify which categorical metadata fields to use for trait reconstruction.

<!-- TODO: can someone please check this section for me? the existing docs were unclear to me -->
To specify this on a per-build basis, add a block like the following to your `my_analyses/<name>/builds.yaml` file:
```yaml
traits:
  north-america: ### build name  
    sampling_bias_correction: 2.5
    columns: ["country_exposure", "division_exposure"] ### traits to reconstruct; must match column names in metadata.tsv
```

This is particularly powerful when travel histories are available.

```yaml
exposure:
  north-america: ## build name  
    trait: "division"
    exposure: "division_exposure"
```

## Labeling clades  

We assign clade labels according to [this schema](XXX).  

Because the exact topology of the tree will vary across runs, clades are defined based on their unique mutations.
These are specified in `default_config/clades.tsv` like so:

```tsv
# clade	gene	site	alt

A1a	ORF3a	251	V
A1a	ORF1a	3606	F
```  


## Keeping a 'Location Build' Up-To-Date

If you are aiming to create a public health build for a state, division, or area of interest, you likely want to keep your analysis up-to-date easily.
If your run contains contextual subsampling (sequences from outside of your focal area), you should first ensure that you regularly download the latest sequences as input, then re-run the build.
This way, you always have a build that reflects the most recent SARS-CoV-2 information.

You should also aim to keep the `ncov` repository updated.
If you've clone the repository from Github, this is done by running `git pull`.
This downloads any changes that we have made to the repoistory to your own computer.
In particular, we add new colors and latitute & longitude information regularly - these should match the new sequences you download, so that you don't need to add this information yourself.

If you don't need to share your build 'profile' with anyone, then it's simple to leave this in the `/profiles` folder.
It won't be changed when you `git pull` for the latest information.

However, if you want to share your build profile, you'll need to adopt one of the following solutions.
First, you can 'fork' the entire `ncov` repository, which means you have your own copy of the repository. 
You can then add your profile files to the repository and anyone else can download them as part of your 'fork' of the repository.
Note that if you do this, you should ensure you `pull` regularly from the original `ncov` repository to keep it up-to-date.

Alternatively, you can create a new repository just to hold your 'profile' files, outside of the `ncov` repository.
You can then share this repository with others, and it's very simple to keep `ncov` up to date, as you don't change it at all.
If doing this, it can be easiest to create a 'profiles' folder and imitate the structure found in the 'profiles' folder within `ncov`, but this isn't required.
Note that to run the build using the profile you'll need still run the `snakemake` command from within the `ncov` repository, but specify that the profile you want is outside that folder.

For the [`south-usa-sarscov2`](https://github.com/emmahodcroft/south-usa-sarscov2/) example, you can see the `south-central ` profile set up in a 'profiles' folder.
To run this, one would call the following from within `ncov`:

```bash
ncov$ snakemake --profile ../south-usa-sarscov2/profiles/south-central/
```
