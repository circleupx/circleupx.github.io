---
title: Python Script to Combine Parquet Files
tags: [Python]
author: "Yunier"
date: "2025-08-20"
description: "Quick Script To Join Parquet or CVS files"
---

Today's blog post is quick and simple, a script I find myself using quite often these days. It takes a collection of CSV or Parquet files and combines them into a single file. 

I find it useful whenever I need to query the same data across multiple files. Beats importing the individual files into a DB client like DBeaver.

```Python
import os
import pandas as pd

# Directory containing your files
dir_path = '/Users/yunier/Downloads'

# Output file name
output_file = os.path.join(dir_path, 'combined_output1.parquet')

# List of files to combine (CSV and Parquet)
files = [f for f in os.listdir(dir_path) if f.endswith('.csv') or f.endswith('.parquet')]

# List to hold DataFrames
dfs = []

for file in files:
    file_path = os.path.join(dir_path, file)
    if file.endswith('.csv'):
        df = pd.read_csv(file_path)
    elif file.endswith('.parquet'):
        df = pd.read_parquet(file_path)
    else:
        continue
    df['source_file'] = file # Track source of data.
    dfs.append(df)

# Combine all DataFrames
if dfs:
    combined_df = pd.concat(dfs, ignore_index=True)
    combined_df.to_parquet(output_file)
    print(f'Combined file saved to: {output_file}')
else:
    print('No CSV or Parquet files found to combine.')
```