# Working example of Unfold BIDS
=> Add an Artifact of one 8-bit subject (maybe downsamples, less channels to save space) using Artifacts.jl (dont ask Bene, but explain it to him how it works ;-)).

TODO: Make one subject into tar file (including BIDS structure)
    Usage: make tar.gz archive of data; upload to repo; make entry in Artifacts.toml with create_artifact, bind_artifact and artifact_hash

```julia
using Unfold, UnfoldBIDS
using DataFrames
```

## Parameter setting
First we need to set some basic Parameter, like the path to our Data:
```julia

# the base bids folder, ...
loc = datadep"EEG8bit"

# the subjects involved, ...
subs = ["001","002","003"];

# the tasks performed, ...
tasks = ["ContinuousVideoGamePlay"];

# the runs ...
runs = ["02"];

# as well as the file type used to store the eeg data, denoted by its file ending (without the dot)
fileEnding = "vhdr";
```


As well as which data we are actually interested in. 

Passing an empty array would in using all 63 channels of the example dataset.\
**Note**: This is no real "choose all channels" behaviour yet. Will be implemented in the future.



```julia
# Channel

interesting_channels = ["Cz"];

# events that should be ignored due to e.g. low sample size
drop_events = ["STATUS","GAME OVER","GAME START"];

# basic time around an event used for epoching and default basis functions (in seconds)
tau = (-0.5, 1.0);
```

And lasty some plotting Parameters:
```julia
# what dataframe columns to use for the x and y axes of the ERP plots
basic_mapping = mapping(:colname_basis => "Time from Event (s)",:estimate => "Estimate (μV)");

# for a larger number of channels, laying out the plots gets incredibly slow, so only execute manually what you need
auto_render_cutoff = 3;
```
## Script work
Now we can start processing our data
```julia


results_r = DataFrame();
results_e = DataFrame();

positions = nothing;

# For now use all channels
# if isempty(interesting_channels) interesting_channels = [1:63;]; end;

# for s in subs
# for t in tasks
# for r in runs

#for testing purposes; also comment in/out the triple "end" at the end of the Using Unfold part
s = subs[1];
t = tasks[1];
r = runs[1];

# fun with bids
currentLoc = loc * "/sub-" * s * "/eeg/sub-" * s * "_task-" * t * "_run-" * r;
```
---
#  Formulas and Functions 1
---
If you've read the main Unfold documentation you know that we will need to specify basis functions for the different events. This can be done like below.\
Note that we will pass an empty dict, which results in all events getting the default values `(@formula( 0 ~ 1 ), firbasis(τ=(-0.4, .8), sfreq=raw.info["sfreq"], name=evt_name))` assigned.\
An example entry would look like this: `"EventA" => (@formula( 0 ~ 1 + Covariate ),firbasis(τ=tau, sfreq=500, name="EventA"))`


The same goes for the epoched based analysis, where the default corresponds to `term(0) ~ term(0) + term(evt_name)`\
Note that for code simplicity all formulas should have at least two terms on the right side, even if one is 0, as in this example: `@formula(0 ~ 0 + EventA)`
```julia
bfDict = Dict{String,Tuple{FormulaTerm, Unfold.BasisFunction}}(
);  #bfDict end

epochedFormulas = [
];  #epochedFormulas end
```

##  Raw Data Processing
Now let's load the raw data and add additional information to get it into a format that Unfold can work with.

```julia
evts_set, evts, raw_data, sfreq = loadRaw(currentLoc, fileEnding, drop_events);

##
chan_types = Dict(:AMMO=>"misc",:HEALTH=>"misc",
                    :PLAYERX=>"misc", :PLAYERY=>"misc",
                    :WALLABOVE=>"misc",:WALLBELOW=>"misc",
                    :CLOSESTENEMY=>"misc",:CLOSESTSTAR=>"misc")

montage = "standard_1020"
##
interesting_channel_names, positions = populateRaw(
    raw_data, chan_types, montage, bfDict, 
    epochedFormulas, interesting_channels, 
    evts_set, evts)

#convert data to μV from Volt to undermine possible underflows in the later calculation
#raw_data does only contain the interesting_channels specified, so unless one specified a stim channel, this simple line is enough
raw_data .*= 10 ^ 6;

if positions === nothing
    global positions = positions_temp;
end

# ---
#  Formulas and Functions 2
# ---

addDefaultEventFormulas!(bfDict,epochedFormulas,evts_set,tau); 

```

## Using Unfold to get ERPS
```julia

# regression-based analysis fits all ERPs at once
uf = fit(UnfoldModel, bfDict, evts, raw_data, eventcolumn="event");
res_r = coeftable(uf)
res_r.channel = [interesting_channel_names[i] for i in res_r.channel];

# epoch-based analysis
res_e = epochedFit(UnfoldLinearModel,epochedFormulas,evts,raw_data,tau,sfreq);
res_e.channel = [interesting_channel_names[i] for i in res_e.channel];

# allow later grouping by subject, task or run
insertcols!(res_r, ([:subject,:task,:run] .=> (s,t,r))...);
insertcols!(res_e, ([:subject,:task,:run] .=> (s,t,r))...);

# depending on changes in newer Julia versions, the globals here might no longer be necessary
global results_r = vcat(results_r,res_r,cols=:union);
global results_e = vcat(results_e,res_e,cols=:union);


```

##  Filtering and grouping the results
```julia

# if covariates were used (Intercept is not the only term), group by term as well, don't compare with epoched results etc.
covariates_used = length(Set(results_r.term))>1;

# different names to keep the different steps around for exploration
# make a grouped DataFrame and ...
if covariates_used
    grouped_r = groupby(results_r,[:basisname,:colname_basis,:channel,:term]);
else
    grouped_r = groupby(results_r,[:basisname,:colname_basis,:channel]);
end
# take the mean over the estimates for each group, but keep the name the same instead of appending _mean to it
combined_r = combine(grouped_r,:estimate => mean => :estimate);

# repeat for epoch-based analysis results
grouped_e = groupby(results_e,[:term,:colname_basis,:channel]);
combined_e = combine(grouped_e,:estimate => mean => :estimate);

# compare both methods
if !covariates_used
    prepped_r = copy(combined_r);
    rename!(prepped_r,:basisname => :basisname_term);
    prepped_r[!,"Analysis Type"] .= :regression_based;

    prepped_e = copy(combined_e);
    rename!(prepped_e,:term => :basisname_term);
    prepped_e[!,"Analysis Type"] .= :epoch_based;

    prepped = vcat(prepped_r,prepped_e);
end
```

## Plotting the Results

```julia
# --> once the script is done, call fg_r, fg_e or fg in the Julia REPL to look at your results in a new window
#  (or the old window incase you already looked at something and didn't close it!)

if length(interesting_channels) <= auto_render_cutoff
    # regression-based ERPs, plot using terms and add legend in case of covariates
    if covariates_used
        fg_r = drawERPs(combined_r, basic_mapping, mapping(col = :basisname, color = :term, row=:channel), addLegend = true);
    else
        fg_r = drawERPs(combined_r, basic_mapping, mapping(col = :basisname, color = :basisname, row=:channel));
    end

    # epoch-based ERPs
    fg_e = drawERPs(combined_e, basic_mapping, mapping(col = :term, color = :term, row = :channel));

    # comparison
    if !covariates_used
        fg = drawERPs(prepped, basic_mapping, mapping(col = :basisname_term, color = Symbol("Analysis Type"), row = :channel),
            addLegend = true);
    end
end

```