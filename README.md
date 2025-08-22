# Narwhals_article_repo

This is the supporting repo for the Medium Article: https://medium.com/@fercv87/an-alternative-to-pandas-narwhals-with-polars-as-backend-2acc3e5dd872

1. Introduction
   
Why is this article worth reading? Because I detail and test a Python library (i.e. Narwhals) that may help you dealing with big datasets while working locally (i.e. with your laptop).

This article aims to answer one simple question: what is the most efficient way to deal with big datasets when you only have your laptop? Of course, there is going to be a point where it is not possible to fit in your laptop´s memory massive datasets (I later explain where I found my laptop´s limit). However, it is valuable to know what alternatives there are to the by-default usage of Pandas.

What is the target audience? This is intended for libraries maintainers and people using Pandas who are facing a roadblock because of the dataset’s size. This is also intended for people mixing different DataFrame tools such as Pandas, Polars, cuDF, DuckDB,… who benefit from writing backend-agnostic DataFrame code.

2. Background: the DataFrame ecosystem
2.1. Current landscape

What Python libraries are there to load a dataset? There are many, the standard is Pandas. We can argue there is a fragmented ecosystem. Below there is a curated table I put together while further researching on this topic.

Press enter or click to view image in full size

The sources used are https://pepy.tech/, for the volume of downloads, and GitHub, for the stars. These datapoints are as of August 2025. I used Claude Sonnet 4 to get a script to programmatically pull the downloads data from pepy.tech stored in GitHub. Apparently, at the time of writing, the alternative pypistats is down.

Press enter or click to view image in full size

2.2. Eager vs lazy execution

Before we move forward, a brief non-technical explanation of what is eager and lazy execution as these concepts are material for this article. Eager execution means operations run immediately and return results (pandas and cuDF are eager). Lazy execution (Polars, Dask) means operations build a plan and only execute when needed, enabling optimizations. Below a summary table for a faster comparison.

Press enter or click to view image in full size

3. Understanding Narwhals
3.1. Why Narwhals?

If you hit a wall using Pandas but still don´t want to discard your code, Narwhals can be the bridge to evolve from eager Pandas to lazy Dask or Polars.

Narwhals decouples DataFrame data processing (joins, aggregations, transformations…) from the underlying engine (e.g. Pandas, Polars, cuDF, PyArrow,…). You can think of it as a translator between an agnostic DataFrame code and the engine running it (technically a compatibility layer). That brings the following benefits (I will try to illustrate them later):

1. No lock-in (bring-your-own-DataFrame): your code doesn’t change, you can switch indistinctively between pandas, Polars,…

2. Performance optionality without code rewrites: that engine switch possibility may be seeking speed: pandas for familiarity, Polars for multi-core speed, cuDF for GPU.

3. Preserves execution model: pandas is eager; Polars/Dask can be lazy. Narwhals doesn’t force a compute unless you ask.

4. Thin by design: Narwhals is a lightweight library developed in pure Python, not a new engine. Overhead is tiny in practice because the heavy lifting still happens in pandas/Polars/cuDF/Arrow.

5. Cleaner code & easier testing: one expression-based pipeline instead of divergent pandas vs Polars branches. You can unit-test the exact same function against multiple backends.

Press enter or click to view image in full size

3.2. What is Narwhals?

In a nutshell, Narwhals is a Python library translating a Polars-like expression API to multiple DataFrame backends (pandas, Polars, cuDF, PyArrow,…). That is as technical as I can get. I would explain it to my son as a translator between different types of tables (i.e. DataFrames).

Press enter or click to view image in full size

Another way to view Narwhals is captured with this diagram.
Press enter or click to view image in full size

This is a high-level diagram of what Narwhals does.
4. Performance testing
Let´s get into the gist with two examples. The first example (full code in GitHub) is an artificial one with sales (2 columns and 200M rows consuming around 3.0GB of memory) & regions dummy DataFrames where I time the performance of using Pandas and Polars as backends of Narwhals. My laptop crashed several times trying with even bigger datasets (1000M and 500M rows), below its specs to put in context the laptop´s capabilities.

Press enter or click to view image in full size

The operations for this first example are a JOIN, addition of a normalised column, a filter, a Group By and an aggregation. Below the key bits of code with Pandas, Polars, and Narhwals with Pandas and Polars as backends.

Press enter or click to view image in full size

Script using Narwhals with Pandas as backend. The original DataFrames were originated with Pandas. 39" of execution time.
Press enter or click to view image in full size

Script using Pandas directly. 34" of execution time.
Press enter or click to view image in full size

Script using Polars directly. 5.7" of execution time.
Press enter or click to view image in full size

Script using Narwhals with Polars backend. The original DataFrames were originated with Polars. 10" of execution time.
What conclusions can we draw from this first testing (recall a pipeline of join + filter + groupby + aggregation for 200M rows, 3GB dataset)?

Pure polars: 7.57 seconds (4.5x faster than pandas)
Narwhals + polars: 10.42 seconds (3.3x faster than pandas, but 38% slower than pure polars)
Pure pandas: 34.20 seconds (baseline)
Narwhals + pandas: 39.02 seconds (+14% overhead)
Narwhals Overhead is consistent:

With pandas backend: ~5 second overhead (14% penalty)
With polars backend: ~3 second overhead (38% penalty)
So then what´s the point of using Narwhals if we observe an overhead? Narwhals value proposition is that enables the same exact code ran on both backends with identical results. Despite overhead, Narwhals+Polars still delivers 3.3x performance improvement over pandas. In summary there is a trade-off of 40% Polars performance for 100% code portability

The second example is with Financial Transactions dataset from Kaggle. I like this dataset as it was created by Caixabank Tech for the 2024 AI Hackathon. One of the files contains transaction data (13,305,915 rows × 12 columns) and consumes 1.2GB of memory. Below the scripts I run to test loading times using Pandas, and Narwhals with Pandas and Polars as backends.

Press enter or click to view image in full size

Press enter or click to view image in full size

Both in the script and the table above we can find similar results as in the first example. Recall the full code is in my GitHub.

5. What are Narwhals challenges and limitations?
Before we jump into the conclusions, I wanted to list what potential drawbacks you may encounter.

Press enter or click to view image in full size

There is one important consideration, Narwhals expresions differ from Pandas expressions. As you may have noticed in the screenshots from the first example, Pandas <merge>, <groupby> are Narhwals/polars <join>, <group_by>. Below there´s a mapping a table between both to bridge the gap. You can find further details in this .xlsx file I put together.

Press enter or click to view image in full size

The careful reader may identify the differences. In a nutshell, there are parameter naming differences (e.g. seed vs random_state, group_by vs groupby), method naming differences (e.g. unique() vs drop_duplicates(), drop_nulls() vs dropna()), expression-based API: Narwhals requires using expressions (nw.col()) for most operations, which are then passed to methods like select(), filter(), with_columns(). Another clarification, Narwhals operations return DataFrames, not Series (even for unique values).

6. Conclusions
Narwhals library is recommendable for tool builders when seeking flexibility and potential performance gains in your data pipelines. The results I got from testing suggest the benefits outweigh the costs in scenarios of multi-backend support and performance tuning.

Other valuable resources related to this topic:

I recently came upon Narwhals watching one of the speeches of London´s PyData event from June, posted in YouTube. I found Marco a very eloquent and structure speaker and that prompted me to investigate more.


After that put some focus on this topic and watched these other two videos:



· https://medium.com/@fareedkhandev/handling-large-datasets-in-python-42f57ccbbd1b

· https://www.geeksforgeeks.org/python/handling-large-datasets-in-python
