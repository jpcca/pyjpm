# `pyjpm`


This repository contains the package codes for the ML4H (2025) submission of *Joint Progression Modeling (JPM): A Probabilistic Framework for Mixed-Pathology Progression*.

## Installation

```bash
pip install pyjpm
```

## Synthetic data generation

At the root directory of this repository, run `bash gen.sh`. Synthetic data will be available at `pyjpm/test/my_data/`.

If you want to know more details of how data is generated, see `pyjpm/generate_data.py`.


## Config.yaml


The [`config.yaml`](config.yaml) file contains the hyper-parameters tunable for data generation and the model run.


## Testing performance


To test the performance of JPM over the synthetic datasets, at the root directory of this repo, run `bash test.sh`.


Results will be available at `pyjpm/test/algo_results/`.


## User your own data


You are more than welcome to use your own data. But please make sure your data conforms to the "tidy format" as shown in `pyjpm/samples/`, where each row is a participant, and there are four columns: participant, biomarker, measurement, diseased. The `diseased` column indicates whether this is a healthy person (False) or a patient (True).


Because of the nature of JPM, we assume you have both the data for the "aggregate ranking" (overall disease), and also for the "partial rankings", i.e., individual diseases.


First, you need to run [`pysaebm`](github.com/jpcca/pysaebm) to get the progression of the individual diseases, in the following format:


```py
dic1 =  {
       "CSFABETA": 2,
       "CSFPTAU": 7,
       "CSFTTAU": 6,
       "EntorhinalNorm": 10,
       "FusiformNorm": 13,
       "HippVolNorm": 11,
       "MidTempNorm": 9,
       "PHC_EXF": 4,
       "PHC_LAN": 1,
       "PHC_MEM": 3,
       "VentricleNorm": 12,
       "WholeBrainNorm": 14,
       "ptau_abeta": 8,
       "ttau_abeta": 5
   }


dic2 = {
       "PHC_EXF": 1,
       "PHC_LAN": 2,
       "PHC_MEM": 3,
       "VentricleNorm": 4,
       "WMHnorm": 6,
       "WholeBrainNorm": 5
   }
```


Suppose you have two of these (i.e., you have two individual diseases), then you can get the `padded_partial_rankings` parameter required by the `pyjpm` package this way:


```py
import numpy


def sort_dict(d):
   return dict(sorted(d.items(), key=lambda x: x[1]))


ad_dic  = sort_dict(dic1)
vad_dic = sort_dict(dic2)


# Union
all_bm   = sorted(set(ad_dic) | set(vad_dic))
str2int  = {bm: i for i, bm in enumerate(all_bm)}


# convert string ranking to integers, with padding
def make_ranking(dic, max_len, mapping):
   # padding
   ranking = -1 * np.ones(max_len, dtype=np.int64)
   for i, bm in enumerate(dic.keys()):
       ranking[i] = mapping[bm]
   return ranking


ranking_len = max(len(ad_dic), len(vad_dic))
ad_ranking  = make_ranking(ad_dic, ranking_len, str2int)
vad_ranking = make_ranking(vad_dic, ranking_len, str2int)


# --- Final stacked rankings ---
padded_partial_rankings = np.vstack([ad_ranking, vad_ranking])
```


Then you can get the results by


```py
from pyjpm import run_mpebm


run_mpebm(
       partial_rankings=padded_partial_rankings,
       bm2int=str2int,
       mp_method=mp_method, # choose from 'Pairwise', 'Mallows_Tau', 'BT', and 'PL'
       save_results=True,
       data_file= data_file, # path to your mixed pathology data. Should conform to the tidy format.
       output_dir='NACC_CO', # path to where you want to store the outputs
       output_folder=mp_method, # subfolder to store outputs. You can skip this, then the results will be in `output_dir`.
       n_iter=20000, # MCMC iterations.
       n_shuffle=2, # number of biomarkers to shuffle in MCMC. default is 2. No need to change unless you know what you are doing.
       burn_in=500, # burn in for plot. We will delete the first 500 MCMC results in the heatmap (but in the traceplot, we have everything starting from iteration 40)
       thinning=1, # no need to change it unless you know what you are doing.
       save_plots = True, # save heatmap and the traceplot
       seed = 42 # set your seed here.
   )
```


After successful running, you'll see four folders: `heatmaps`, `records`, `results`, `traceplots`.


For the method of `Mallows_Tau`, you can set the `mallows_temperature` parameter in `run_mpebm`. Also, if you are testing synthetic data where you know true progression and the true participant disease stages, you can set the `true_order_dict` and the `true_stages` parameters. Note that the `true_order_dict` is a dictionary where the key is the biomarker (in string) and the value is the position starting from 1.


For details, see [`pyjpm/test/test.py`](pyjpm/test/test.py).




## Change Log


- 2025-07-16 (V 0.0.6)
   - Added `mp_method = BT` in `algorithm.py` and `run.py`.
   - Added `PL`.
   - Fixed the bug in `generate_data.py`.


- 2025-07-19 (V 0.0.7)
   - Added the class of `PlackettLuce`.


- 2025-07-20 (V 0.0.15)
   - Updated the definition and implementation of conflict and certainty.
   - Make sure `data` folder exists after uploading to pypi.
   - Make sure `fixed_biomarker_order = True` if we use mixed pathology in data generation.
   - Fixed a bug in the calculation of `conflict`.
   - Make sure the `algorithm.py` is using the correct energy calculation functions.
   - Added entropy and certainty calculation in `MCMC` sampler.
   - Make sure in `generate_data.py`, `certainty` is calculated based upon the `mp_method`.
   - Make sure in `generate_data.py`, we can tweak the parameter of `mcmc_iterations`, otherwise it will be super slow. This is because the time complexity is `mcmc_iterations * sample_count`.
   - Tested obtaining `ordering_array` from separate disease data files. Made some modifications in `algorithm.py` to allow this.


- 2025-07-23 (V 0.0.16 -- didn't push to Pypi)
   - Implemented the conflict version of using only discordant pairs.


- 2025-07-25 (V 0.0.16)
   - Updated `algorithm.py` to reflect changes in the class of `PlackettLuce`.


- 2025-07-26 (V 0.0.17)
   - Updated `generate_data.py` to skip calculating certainty and conflict if `sample_count <= 1`.
 - 2025-08-04 (V 0.1.7)
   - Use `fastsaebm` codes.
   - Finished testing and data generation.
   - With `m_{variant}` when number of repetition is 1.
   - Fixed overflow bug in `prob_accept=min(1.0, np.exp(current_energy - new_energy))`.
 - 2025-08-04 (V 0.2.9)
   - Implemented the new certainty measure. 
   - Used the same `rng` all throughout in generating data.
   - Added `save_details` to `run.py`.
   - Solved the logic bug of `save_details` and `save_results`.
   - Ensured the randomness again in `generate_data.py`.
 - 2025-08-09 (V 0.3.1)
   - Used 15 biomarkers.
   - Dynamically adjust dirichlet multinomial alpha array based on the number of biomarkers.
 - 2025-08-10 (V 0.3.3)
   - Use numpy and numba (whenever possible) in `mp_utils.py`.


- 2025-08-11 (V 0.3.6)
   - Updated numba version
- 2025-08-12 (V 0.3.9)
   - Keep improving the numba version. Now it's faster.
   - Include MCMC in PL sampling as well.


- 2025-08-11 (V 0.4.0)
   - Add RMJ distance mallows.
 - 2025-08-16 (V 0.4.2)
   - Try all `njit` in `mp_utils.py`. I want to test it on CHTC.


- 2025-08-17 (V 0.4.4)
   - I know using `np.random` is not helpful in `shuffle_order` func. Change back to the slow version.


- 2025-08-18 (V 0.4.10)
   - Try to use rng in the func of `obtain_affected_and_non_clusters`. 
   - In mallows, use BT for central ranking sampling.
   - Added `theta = 100` in mallows.
   - Added `mp_method = 'random'` in `generate_data.py`.
   - Removed `recodes` folder when running mp-ebm.


- 2025-08-19 (V 0.4.19)
   - Corrected an error: in data generation, for experiment 9, the noise_std should be max_length * noise_std_parameter rather than its square root. This is important because after using square root, the noise_std in fact becomes larger, not smaller. For example, in our example where N = 10, the noise_std should be N*0.05 = 0.5, but after square root, it becomes 0.7. If N = 4, then std should be 02, but becomes 0.45 after square root.
   - Added the `temperature` parameter for mallows sampling.
   - Reuse parameters inferred from individual diseases.
   - Used biomarkers and theta/phi params obtained from NACC data analysis.


- 2025-08-20 (v 0.5.1)
   - Used biomarkers and their theta/phi from both ADNI only.
   - Changed `'random'` to `'Random'` in `generate_data.py`.
   - Randomly choose two floats for new theta parameters for overlapping biomarkers.


- 2025-08-21 (V 0.5.4)
   - Try only 12 biomarkers for params.
   - Try 18 biomarkers for params.
   - Add random perturbations to overlapped biomarkers params.


- 2025-08-22 (V 0.5.6)
   - Try scaling factor for energy in `mh.py`


- 2025-08-23 (V 0.5.9)
   - Test different energy influence.
   - `use_scaling`
- 205-08-25 (V 0.6.0)
  - Test `percentile`.
 - 205-08-26 (V 0.6.2)
 - Added analysis about `alignment` and `effect_size`.




- 2025-08-27 (V 0.6.7)
 - Added `energy_prior` and `model_prior`.
 - Mapped `energy_prior` to `mallows_temperature`.


- 2025-08-28 (V 0.6.9)
 - Removed `energy_prior`. Only use the model `calibration`.


- 2025-08-32 (V 0.7.3)
 - Modified the data generation just like in subtypes.


- 2025-09-01 (V 0.7.4)
 - Removed the forcing range of `event times`.


- 2025-09-03 (V 0.7.5)
 - Added `save_data` boolean to `generate_data.py`.


- 2025-09-04 (V 0.8.0)
 - Removed `calibration`. We cannot use it.
 - Aligned with how I get staging with `pysaebm`: completely blind, not even using a healthy ratio and the learned stage prior. Only use the theta/phi.
 - Modified what to return in `run.py`.


- 2025-09-09 (V 0.8.1)
 - Added plots back.
 - 2025-09-21 (V 0.8.2)
   - Removed `iteration >= burn_in` when updating best_*.
 - 2025-10-08 (V 0.8.4)
   - Used soft counts for conjugate prior updates.
 - 2025-10-09 (V 0.8.6)
   - Update the non-normal distribution parameters.


- 2025-11-09 (V 0.8.8)
   - Changed the pkg name from `pympebm` to `pyjpm`.
