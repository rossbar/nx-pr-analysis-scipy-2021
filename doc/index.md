---
jupytext:
  notebook_metadata_filter: all
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.11.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
language_info:
  codemirror_mode:
    name: ipython
    version: 3
  file_extension: .py
  mimetype: text/x-python
  name: python
  nbconvert_exporter: python
  pygments_lexer: ipython3
  version: 3.7.4
---

# NetworkX Pull Request Analysis

Some basic analysis of the development history of NetworkX vis-a-vis pull
requests.

```{code-cell}
import numpy as np
import matplotlib.pyplot as plt
import json

# Some matplotlib settings
mpl_params = {
    "axes.titlesize": 24,
    "axes.labelsize": 20,
    "xtick.labelsize": 16,
    "ytick.labelsize": 16,
    "lines.linewidth": 4,
    "legend.fontsize": 16,
    "figure.figsize": (16, 12),
}
plt.rcParams.update(mpl_params)

import warnings
warnings.simplefilter("ignore")

fname = "../data/prs.json"
with open(fname, 'r') as fh:
    data = json.loads(fh.read())
```

## Merged pull requests over time

```{code-cell}
---
tags: [hide-input]
---
merged_prs = [d for d in data if d['node']['state'] == 'MERGED']
merge_dates = np.array([r['node']['mergedAt'] for r in merged_prs], dtype=np.datetime64)
binsize = np.timedelta64(14, 'D')
date_bins = np.arange(merge_dates[0], merge_dates[-1], binsize)
h, be = np.histogram(merge_dates, date_bins)
bc = be[:-1] + binsize / 2
smoothing_interval = 8  # in units of bin-width

fig, ax = plt.subplots()
ax.bar(bc, h, width=binsize, label="Raw")
ax.plot(
    bc,
    np.convolve(h, np.ones(smoothing_interval), 'same') / smoothing_interval,
    label=f"{binsize * smoothing_interval} moving average",
    color='tab:orange',
)
fig.autofmt_xdate()

ax.set_title('Merged PRs over time')
ax.set_xlabel('Time')
ax.set_ylabel(f'# Merged PRs / {binsize} interval')
ax.legend();
```

## Number of First-Time Contributors over time

```{code-cell}
---
tags: [hide-input]
---
first_time_contributor = []
prev_contrib = set()
for record in merged_prs:
    try:
        author = record['node']['author']['login']
    except TypeError:  # Author no longer has GitHub account
        first_time_contributor.append(None)
        continue
    if author not in prev_contrib:
        first_time_contributor.append(True)
        prev_contrib.add(author)
    else:
        first_time_contributor.append(False)
# Object dtype for handling None
first_time_contributor = np.array(first_time_contributor, dtype=object)
# Focus on first time contributors
ftc_mask = first_time_contributor == True
ftc_dates = merge_dates[ftc_mask]

binsize = np.timedelta64(30, 'D')
date_bins = np.arange(merge_dates[0], merge_dates[-1], binsize)
h, be = np.histogram(ftc_dates, date_bins)
bc = be[:-1] + binsize / 2
smoothing_interval = 3  # in units of bin width

fig, ax = plt.subplots()
ax.bar(bc, h, width=binsize, label="Raw")
ax.plot(
    bc, np.convolve(h, np.ones(smoothing_interval), 'same') / smoothing_interval,
    label=f"{binsize * smoothing_interval} moving average",
    color="tab:orange",
)
fig.autofmt_xdate()

ax.set_title('First-time contributors')
ax.set_xlabel('Time')
ax.set_ylabel('# Merged PRs from FTC / month')
ax.legend()

plt.show()
```
