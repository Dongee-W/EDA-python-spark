# Seaborn for EDA
See `seaborn.ipynb`

# Jupyter Setups
```
import seaborn as sns

%matplotlib inline

the_economist = ['#90353B', '#1A476F', '#008BBC', '#9C8847']
flatui = ["#0099cc", "#993333"]
sns.palplot(sns.color_palette(flatui))
```

# Sample Datasets
```
tips = sns.load_dataset('tips')
flights = sns.load_dataset('flights')
iris = sns.load_dataset('iris')
diamonds = sns.load_dataset('diamonds')

category = tips.select_dtypes('category')
numeric = tips._get_numeric_data()
```

# Plot settings and saving figures
Plot size
```
a4_dims = (11.7, 6)
fig, ax = plt.subplots(figsize=a4_dims)
sns.distplot(ax=ax, a=tips['tip'],bins=30,color="y")
```
Saving figures
```
fig.savefig("test.png")
```

# Univariate plots
## Categorical
Count plot
```
sns.countplot(x='sex', data=tips, palette=sns.color_palette(the_economist))
```
## Numerical
Distribution plot
```
sns.distplot(tips['tip'],bins=30,color="y")
```

# Bivariate plots
## Categorical & categorical
Simple - bucket + sub-bucket
```
g = sns.FacetGrid(tips, col="time")
g.map(sns.countplot, 'sex', palette=sns.color_palette(the_economist))
```
Simple - count matrix
```
from matplotlib.colors import ListedColormap
cmap = ListedColormap(sns.cubehelix_palette(20).as_hex())
sns.heatmap(pd.crosstab(tips['sex'], tips['time']), annot=True, cmap=cmap)
```
Compact
```
fig = plt.figure(figsize=(16, 18))
fig.subplots_adjust(hspace=0.2, wspace=0.2)
for i in range(1, category.columns.size + 1):
    for j in range(1, category.columns.size + 1):
        index = 4 * (i-1) + j
        ax = fig.add_subplot(4, 4, index)
        if (i != j):
            sns.heatmap(pd.crosstab(category.iloc[:,i-1], category.iloc[:,j-1]), annot=True, cmap=cmap)
        else:
            sns.countplot(x=category.columns[i-1], data=category, palette=sns.cubehelix_palette(20))
```
## Categorical & numerical
Simple - bar
```
sns.barplot(x='sex', y='total_bill', data=tips,palette=sns.color_palette(the_economist))
```
Simple - boxplot
```
sns.boxplot(x='day',y='total_bill', data=tips, palette=sns.color_palette(the_economist))
```
Simple - stripplot
```
sns.stripplot(x='day',y='total_bill',data=tips,palette=sns.color_palette(the_economist), jitter=True)
```
Compact
```
g = sns.PairGrid(tips, y_vars=numeric.columns, x_vars=category.columns)
g.map(sns.barplot, palette=sns.color_palette(the_economist))
```
## Numerical & numerical
Simple - joint distribution
```
sns.jointplot(x='total_bill', y='tip', data=tips, color="y",kind='reg')

# For large amount of data
with sns.axes_style("white"):
    sns.jointplot(x='total_bill', y='tip', data=tips, kind="hex", color="k")

# For large amount of data
sns.jointplot(x='total_bill', y='tip', data=tips, kind="kde");
```
Simple - regression plot
```
sns.lmplot(x='total_bill',y='tip',data=tips,hue='sex',
           palette=sns.color_palette(the_economist),markers=['o','x'])
```
Compact - detailed
```
sns.pairplot(data=tips, hue='sex', palette=sns.color_palette(the_economist))
```
Compact - correlation heatmap 1
```
tc = tips.corr()

from matplotlib.colors import ListedColormap
cmap = ListedColormap(sns.cubehelix_palette(20).as_hex())
sns.heatmap(tc, cmap=cmap,annot=True,linecolor='#FFFFFF')
```
Compact - correlation heatmap 2
```
dc = diamonds.corr()
sns.heatmap(dc, annot=True, cmap='plasma')
```

# Others
```
fm = flights.pivot_table(index='month',columns='year',values='passengers')
sns.heatmap(fm, cmap='plasma')
sns.clustermap(fm, cmap='plasma')
```