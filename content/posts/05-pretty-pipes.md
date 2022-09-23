---
title: "Pretty Pipes for Perusable (and Reusable) Pandas Procedures"
date: 2022-09-22T16:29:13-04:00
draft: false
categories: 
    - python
tags: 
    - pandas
    - python
    - functional
    - pipe
---

Pandas is a fantastic library for data analysis, but its syntax can be a bit jarring to the unfamiliar user, especially one coming from the R `tidyverse` [ecosystem](https://www.tidyverse.org/) where the `%>%` (pipe) operator makes method-chaining powerful and preferred for most operations. It turns out that a similar syntax is totally possible in `pandas` with the `pipe` and `assign` methods! With these, you can make your `pandas` code much more readable and reusable.

From the `pandas` docs, the `pipe` [method]((https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.pipe.html)) lets you write:

```python
(df.pipe(h)
   .pipe(g, arg1=a)
   .pipe(func, arg2=b, arg3=c)
) 
```

instead of:

```python
func(g(h(df), arg1=a), arg2=b, arg3=c)  
```

By default, `pipe` passes the dataframe to the first argument of the function along with any specified keyword arguments. This lets you write any function that receives and returns a dataframe and add it into a series of pipes.

Let's dive into an example.

Frequently, data that was entered by hand has small variations (typos, style changes, etc.) that need to be ironed out before the data can be used for modeling. In my toy example based on some real data I've received below, I'd like to predict themes from service reviews that were labeled by hand.

```python
import pandas as pd

raw = pd.DataFrame(
    {
        'id': ['001', '002', '003', '004'], 
        'label': [
            'great  Communication', 
            'attention to detail', 
            'great communication ', 
            'Attention to Detail'
        ]
    }
)
```

As you can see, the labels are mostly right barring some issues with extraneous spaces and varying capitalization. As a first pass, I could write the following code to standardize the themes to snake case:

```python
raw['label'] = (
        raw['label']
        .str.strip()
        .str.lower()
        .str.replace(r'\W+', '_', regex=True)
    )
```

This code works, but it's tightly coupled to the column name, which is written in two places. It's also not immediately obvious why I'm going to the trouble without a comment or prior knowledge of the data. Better to wrap it in a function!

```python
def convert_to_snake_case(
    dataframe: pd.DataFrame, columns: List[str]
) -> pd.DataFrame:
    """Convert values in string columns in a dataframe to snake case, handling extraneous spacing and capitalization.
    """
    assert set(columns).issubset(
        set(dataframe.columns)
    ), "At least one column is missing."

    assert all(
        pd.api.types.is_string_dtype(dataframe[column]) 
        for column in columns
    ), "All columns must be of string dtype."

    return dataframe.assign(
        **{
            column: (
                dataframe[column]
                .str.strip()
                .str.lower()
                .str.replace(r'\W+', '_', regex=True)
            )
            for column in columns
        }
    )
```

I can avoid using the bracket syntax to modify columns by passing keyword arguments (here, using the ** expansion syntax with a dictionary comprehension) to `assign` to redefine all of the columns. The `assign` method returns a dataframe, so could I chain any dataframe method after it. Since I've specified that this function takes a `DataFrame` as its first argument, it can be easily used in a pipe.[^1]

```python
tabs = (
    old
    .pipe(convert_to_snake_case, columns = ['label'])
    .value_counts()
)
```

One caveat to this approach is that the tracebacks resulting from errors occuring in the piped function can be a bit more difficult to parse since the proximate cause of the error will be `pandas` error-handling code instead of the function call. For this reason, I like to include asserts that guard against common mistakes in the functions that I might pipe to make them easier to debug. Hence, the `assert` calls above checking that every column in `columns` exists and has a string data type.

Extracting data preprocessing components into functions has a nice payoff when combined in a series of pipes. For example, you might structure your code like this:

```python
    
def load_raw() -> pd.DataFrame:
    return pd.read_csv(...) # e.g.

def remove_invalid_columns(data: pd.DataFrame) -> pd.DataFrame:
    ...

def remove_invalid_rows(data: pd.DataFrame) -> pd.DataFrame:
    ...

def convert_to_snake_case(data: pd.DataFrame, columns: List[str]) -> pd.DataFrame:
    ...

def run_all(data: pd.DataFrame) -> pd.DataFrame:

    return (
        .pipe(self.remove_invalid_columns)
        .pipe(self.remove_invalid_rows)
        .pipe(self.convert_to_snake_case, columns = ["label"])
    )

```

Of course, all of this is possible with nested function calls or intermediate variables in the absence of `pipe`, but I find the piped functions above easier to follow without the visual noise. Plus, it's easier to take the time to reorganize data munging code into component functions when the final `run_all` function looks so nice!

[^1]: You can pass a (function, dataframe-argument-name) tuple as the first argument to `pipe` if the dataframe argument is later in the function signature.
