### **Fusion Transcript (FT) Raw Data Exploration**

#### **Introduction**

This notebook details the processes (semi-automated) done to further process the raw output files from Arriba and FusionCatcher fusion transcript callers. 

1. Run the `pypolars-process-ft-tsv.py` script to generate fusion transcript list from Arriba and FusionCatcher output files. The script takes a mandatory input of path to the directory where sample-specific fusion call output files from Arriba or FusionCatcher are stored as the first argument, and the specific string that is used to identify tool name (`arr` for Arriba fusion transcript call output file prefix, for instance). 

	For example:
	> ``` pypolars-process-ft-tsv.py data/FTmyBRCAs_raw/Arriba arr ```

	Do the same for the FusionCatcher raw output files, as well as the same Arriba and FusionCatcher output files generated from the processing 113 TCGA-Normals (to use as a panel of normals for FT filtering).

2. Then, load up the two datasets on Jupyter Notebook and concatenate the dataframes together so that Arriba+FusionCatcher unfiltered FT data are combined into one data table and saved in one `.parquet` and `.tsv` file. Do the same for the `TCGANormals` panel of normals.


```python
# first, import packages
import polars as pl
import pandas as pd

from itables import init_notebook_mode, show
init_notebook_mode(all_interactive=True)
import itables.options as opt
opt.maxBytes = "250KB"
```


```python
# load up MyBrCa datasets
arr_mdf = pl.scan_parquet('output/MyBrCa/Arriba-FT-all-unfilt-list-v2.parquet')
fc_mdf = pl.scan_parquet('output/MyBrCa/FusionCatcher-FT-all-unfilt-list-v2.parquet')

# now load TCGANormals
arr_norms_mdf = pl.scan_parquet('output/TCGANormals/Arriba-Normal-FT-all-unfilt-list-v2.parquet')
fc_norms_mdf = pl.scan_parquet('output/TCGANormals/FusionCatcher-Normal-FT-all-unfilt-list-v2.parquet')
```

##### **Dataset 1A** (MyBrCa): Arriba unfiltered


```python
from IPython.display import display, HTML

display(HTML("Arriba MyBrCa datatable dimension: " + f"<b>{arr_mdf.collect().shape}</b>"))

pl.DataFrame.to_pandas(arr_mdf.collect(), use_pyarrow_extension_array=True)
```

##### **Dataset 1B** (MyBrCa): FusionCatcher unfiltered


```python
from IPython.display import display, HTML

display(HTML("FusionCatcher MyBrCa datatable dimension: " + f"<b>{fc_mdf.collect().shape}</b>"))

pl.DataFrame.to_pandas(fc_mdf.collect(), use_pyarrow_extension_array=True)
```

##### **Dataset 2A** (TCGA Normals): Arriba unfiltered


```python
from IPython.display import display, HTML

display(HTML("Arriba TCGA-Normals datatable dimension: " + f"<b>{arr_norms_mdf.collect().shape}</b>"))

pl.DataFrame.to_pandas(arr_norms_mdf.collect(), use_pyarrow_extension_array=True)
```

##### **Dataset 2B** (TCGA Normals): FusionCatcher unfiltered


```python
from IPython.display import display, HTML

display(HTML("FusionCatcher TCGA-Normals datatable dimension: " + f"<b>{fc_norms_mdf.collect().shape}</b>"))

pl.DataFrame.to_pandas(fc_norms_mdf.collect(), use_pyarrow_extension_array=True)
```

#### **Concatenate Arriba and FusionCatcher Datasets**

Now, we can merge the two dataframes into one masterFrame for each cohort data (MyBrCa & TCGA panel of normals) using Polars' `concat` (vertical concatenation is the default, where two dataframes sharing the exact same columns would be joined together, adding all rows of dataframe 1 and 2 vertically).


```python
%%capture --no-stdout --no-display

joined_df = pl.concat(
    [
        arr_mdf.collect(),
        fc_mdf.collect()
    ]
)
from IPython.display import display, HTML

display(HTML("Concatenated MyBrCa Arriba+FusionCatcher datatable dimension: " + f"<b>{joined_df.shape}</b>"))

pl.DataFrame.to_pandas(joined_df, use_pyarrow_extension_array=True)
```


```python
# save joined df to files

### UNCOMMENT TO SAVE
# joined_df.write_csv('output/MyBrCa/Arr_FC-concat-FT-all-unfilt-list-v2.tsv', separator='\t')

# joined_df.write_parquet('output/MyBrCa/Arr_FC-concat-FT-all-unfilt-list-v2.parquet')
```

Do the same with the TCGA panel of normal FTs.


```python
%%capture --no-stdout --no-display

joined_norms_df = pl.concat(
    [
        arr_norms_mdf.collect(),
        fc_norms_mdf.collect()
    ]
)
from IPython.display import display, HTML

display(HTML("Concatenated TCGA Normals Arriba+FusionCatcher datatable dimension: " + f"<b>{joined_norms_df.shape}</b>"))

pl.DataFrame.to_pandas(joined_norms_df, use_pyarrow_extension_array=True)
```


```python
# save joined df to files

### UNCOMMENT TO SAVE
# joined_norms_df.write_csv('output/TCGANormals/Arr_FC-Normals-concat-FT-all-unfilt-list-v2.tsv', separator='\t')

# joined_norms_df.write_parquet('output/TCGANormals/Arr_FC-Normals-concat-FT-all-unfilt-list-v2.parquet')
```

#### **Filter MyBrCa Merged Datatable using Panel of Normals**
Now we can filter the unfiltered FT datatable by discarding those that are present in TCGA Normal datatable.



```python
# first load the parquet files
# load up MyBrCa datasets
mybrca_ccdf = pl.scan_parquet('output/MyBrCa/Arr_FC-concat-FT-all-unfilt-list-v2.parquet')

# now load TCGANormals
norms_ccdf = pl.scan_parquet('output/TCGANormals/Arr_FC-Normals-concat-FT-all-unfilt-list-v2.parquet')

```


```python
mybrca_ccdf_pan = pl.DataFrame.to_pandas(mybrca_ccdf.collect(), use_pyarrow_extension_array=True)

show(mybrca_ccdf_pan)
```


```python
norms_ccdf_pan = pl.DataFrame.to_pandas(norms_ccdf.collect(), use_pyarrow_extension_array=True)

show(norms_ccdf_pan)
```

Use Polars' `filter` expression with `is_in` and the negation `~` to keep only unique rows for column `breakpointID` in MyBrCa dataframe that are NOT in the `breakpointID` in TCGANormals dataframe. 


```python
normfilt_mybrca_ccdf = mybrca_ccdf.collect().filter(~pl.col('breakpointID').is_in(norms_ccdf.collect()['breakpointID']))

show(normfilt_mybrca_ccdf, maxBytes=0)
```

Use `group_by` and `n_unique()` in Polars to create a count table for unique `breakpointID` and how "shared" it is across our MyBrCa cohort. 


```python
# test on toy df
# df = pl.DataFrame(
#     {
#         "a": [1, 1, 2, 3, 4, 5],
#         "b": [0.5, 0.5, 1.0, 2.0, 3.0, 3.0],
#         "c": [True, True, True, False, True, True],
#     }
# )
# print(df)
# print(df.select(pl.col(["a", "b"])))
# print(df.select(pl.col(["a", "b"])).group_by("b").n_unique())
```

Here, we subset the dataframe into just `breakpointID` and `sampleID` and then use `group_by` on the `breakpointID` column, then counting number of unique occurences of each unique `breakpointID` in the `sampleID` column. 

This would return a count of unique samples (patients) one particular unique breakpoint appears in. I call this the `sharednessDegree`.

> **NOTE:** This is the best way to address miscounting breakpoints that appear in multiple rows due to differences in gene naming but they are only seen in one sample. Using other counting strategies such as window function (`.over` method) will count these duplicate rows as separate entities when in reality they are the same breakpoint seen in just one patient.


```python
show(normfilt_mybrca_ccdf.select(pl.col(["breakpointID", "sampleID"])).group_by("breakpointID").n_unique().rename({"sampleID": "sharednessDegree"}), maxBytes=0) 
```

#### **Plotting the Sharedness Degree**

We have used Polars to easily group and count the number of patients sharing a particular breakpoint ID for each unique breakpoint ID as above, let's formalize that again by using Pandas instead.

First, subset the filtered dataframe to just the two columns we are interested in using Polars, but this time prepend the "P" string to all values of the `sampleID` column.


```python
bp_sample_array = normfilt_mybrca_ccdf.select(
    pl.col("breakpointID"),
    pl.concat_str(pl.lit("P"), pl.col("sampleID")).alias("sampleID")
)
```

Then we convert to pandas for easier visualization.


```python
bpsample_pdf = bp_sample_array.to_pandas()
bpsample_pdf
```

Due to the annotation redundancy in `fusionGeneID` column in the original df, we now have rows in `breakpointID` and `sampleID` that are repeated (i.e. `6:36132629-17:44965446	P1` as seen above). Let's filter these out, as they represent the same putative FT.


```python
# Drop duplicates based on both columns
bpsample_pdf_unique = bpsample_pdf.drop_duplicates()

# see how many duplicates were removed
print("Original number of rows:", len(bpsample_pdf))
print("Number of rows after removing duplicates:", len(bpsample_pdf_unique))
print("Number of duplicates removed:", len(bpsample_pdf) - len(bpsample_pdf_unique))
```


```python
bpsample_pdf_unique
```

Now group by each unique `breakpointID` and count how many `sampleID` is associated with this breakpoint (*sharedness degree*).


```python
# Group by breakpointID and count unique sampleIDs
breakpoint_counts = bpsample_pdf_unique.groupby('breakpointID')['sampleID'].nunique().reset_index()

# Rename the column for clarity
breakpoint_counts = breakpoint_counts.rename(columns={'sampleID': 'sharednessDegree'})

show(breakpoint_counts, maxBytes=0)

```

Then we can count the number of unique `breakpointID`s for each `sharednessDegree` value.


```python
# Count the number of unique breakpointIDs for each sharednessDegree value
sharedness_counts = (
    breakpoint_counts
    .groupby('sharednessDegree')
    .agg(
        unique_bp_count=('breakpointID', 'nunique')
    )
    .reset_index()
    .sort_values('sharednessDegree')
)

sharedness_counts
```


```python
import matplotlib.pyplot as plt
import seaborn as sns

# Create the bar plot
plt.figure(figsize=(10, 8), dpi=300)
sns.barplot(x=sharedness_counts['sharednessDegree'],y=sharedness_counts['unique_bp_count'], color='steelblue')

# Add value labels on top of the bars
for i, v in enumerate(sharedness_counts['unique_bp_count']):
    plt.text(i, v, str(v), color='black', ha='center', fontweight='bold', fontsize=8)
    
# Set labels and title
plt.xlabel('Sharedness Degree (Number of Patients A Unique FT Is Observed)')
plt.ylabel('Count of Unique FTs')
plt.title('Frequency of Unique Tumor-Specific FTs by Sharedness Degree')

# Rotate x-axis labels for better readability
plt.xticks(rotation=90)

plt.show()
```

### Using Graph Theory to Investigate Bipartite Relationship

We can use graph theory to explore the underlying bipartite network between unique `breakpointID` and `sampleID`.

Create a complex class called `NetworkAnalyzer` to do graph network analysis between `breakpointID` and `sampleID`. 


```python
# import numpy as np
# import networkx as nx
# import plotly.graph_objects as go
# from plotly.subplots import make_subplots
# from scipy.spatial.distance import pdist, squareform
# from networkx.algorithms.bipartite import density as bipartite_density

# class NetworkAnalyzer:
# 	def __init__(self, df: pl.DataFrame, patient_col: str, breakpoint_col: str):
# 		self.df = df
# 		self.patient_col = patient_col
# 		self.breakpoint_col = breakpoint_col

# 		# Get unique sets
# 		self.patients = sorted(df[patient_col].unique().to_list())
# 		self.breakpoints = sorted(df[breakpoint_col].unique().to_list())

# 		# Create adjacency matrix
# 		self.adj_matrix = self._create_adjacency_matrix()

# 		# Calculate network metrics
# 		self._calculate_metrics()

# 	def _create_adjacency_matrix(self) -> np.ndarray:
# 		matrix = np.zeros((len(self.patients), len(self.breakpoints)))
# 		connections = self.df.group_by(self.patient_col).agg(
# 			pl.col(self.breakpoint_col).alias('breakpoints')
# 		).to_dict(as_series=False)

# 		patient_idx = {p: i for i, p in enumerate(self.patients)}
# 		breakpoint_idx = {b: i for i, b in enumerate(self.breakpoints)}

# 		for i, patient in enumerate(connections[self.patient_col]):
# 			for bp in connections['breakpoints'][i]:
# 				matrix[patient_idx[patient]][breakpoint_idx[bp]] = 1

# 		return matrix

# 	def _calculate_metrics(self):
# 		self.patient_degrees = np.sum(self.adj_matrix, axis=1)
# 		self.breakpoint_degrees = np.sum(self.adj_matrix, axis=0)
# 		self.patient_similarity = squareform(pdist(self.adj_matrix, metric='jaccard'))
# 		self.breakpoint_similarity = squareform(pdist(self.adj_matrix.T, metric='jaccard'))

# 		# Create bipartite graph manually
# 		G = nx.Graph()
# 		# Add patient nodes
# 		G.add_nodes_from(self.patients, bipartite=0)
# 		# Add breakpoint nodes
# 		G.add_nodes_from(self.breakpoints, bipartite=1)
# 		# Add edges from adjacency matrix
# 		for i, patient in enumerate(self.patients):
# 			for j, breakpoint in enumerate(self.breakpoints):
# 				if self.adj_matrix[i, j]:
# 					G.add_edge(patient, breakpoint)

# 		centrality = nx.degree_centrality(G)
# 		self.breakpoint_centrality = [centrality[bp] for bp in self.breakpoints]
# 		self.density = bipartite_density(G, self.breakpoints)

# 	def create_adjacency_matrix_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
# 		"""
# 		Create a standalone plot of the patient-breakpoint adjacency matrix.
# 		If `top_bins` is provided, it should be a list of breakpoint indexes representing the top 0.1% of breakpoints by degree.
# 		If `bottom_bins` is provided, it should be a list of breakpoint indexes representing the bottom 0.1% of breakpoints by degree.
# 		"""
# 		if top_bins:
# 			top_breakpoints = [self.breakpoints[i] for i in top_bins]
# 			top_matrix = self.adj_matrix[:, top_bins]

# 			fig = go.Figure(
# 				data=go.Heatmap(
# 					z=top_matrix,
# 					x=top_breakpoints,
# 					y=self.patients,
# 					colorscale="Blues",
# 					showscale=True,
# 					hoverongaps=False,
# 					hoverinfo='text',
# 					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if top_matrix[i][j] else 'No'}"
# 							for j, b in enumerate(top_breakpoints)]
# 							for i, p in enumerate(self.patients)],
# 					colorbar=dict(title="Connection"),
# 					name="Top Breakpoints"
# 				)
# 			)

# 			fig.update_layout(
# 				height=800,
# 				width=1000,
# 				title=dict(
# 					text="Top 0.1% Breakpoints - Adjacency Matrix",
# 					x=0.5,
# 					y=0.95,
# 					font=dict(size=18)
# 				),
# 				xaxis_title="Breakpoints",
# 				yaxis_title="Patients",
# 				template="simple_white"
# 			)

# 		if bottom_bins:
# 			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
# 			bottom_matrix = self.adj_matrix[:, bottom_bins]

# 			fig = go.Figure(
# 				data=go.Heatmap(
# 					z=bottom_matrix,
# 					x=bottom_breakpoints,
# 					y=self.patients,
# 					colorscale="Blues",
# 					showscale=True,
# 					hoverongaps=False,
# 					hoverinfo='text',
# 					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if bottom_matrix[i][j] else 'No'}"
# 							for j, b in enumerate(bottom_breakpoints)]
# 							for i, p in enumerate(self.patients)],
# 					colorbar=dict(title="Connection"),
# 					name="Bottom Breakpoints"
# 				)
# 			)

# 			fig.update_layout(
# 				height=800,
# 				width=1000,
# 				title=dict(
# 					text="Bottom 0.1% Breakpoints - Adjacency Matrix",
# 					x=0.5,
# 					y=0.95,
# 					font=dict(size=18)
# 				),
# 				xaxis_title="Breakpoints",
# 				yaxis_title="Patients",
# 				template="simple_white"
# 			)

# 		return fig

# 	def create_degree_distribution_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
# 		"""
# 		Create a standalone plot of the breakpoint degree distribution.
# 		If `top_bins` is provided, it should be a list of breakpoint indexes representing the top 0.1% of breakpoints by degree.
# 		If `bottom_bins` is provided, it should be a list of breakpoint indexes representing the bottom 0.1% of breakpoints by degree.
# 		"""
# 		if top_bins:
# 			top_breakpoints = [self.breakpoints[i] for i in top_bins]
# 			top_degrees = [self.breakpoint_degrees[i] for i in top_bins]

# 			fig = go.Figure(
# 				data=go.Bar(
# 					x=top_breakpoints,
# 					y=top_degrees,
# 					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" for bp, deg in zip(top_breakpoints, top_degrees)],
# 					hoverinfo='text',
# 					marker_color='rgb(158,202,225)',
# 					marker_line_color='rgb(8,48,107)',
# 					marker_line_width=1.5,
# 					name="Top Breakpoints"
# 				)
# 			)

# 			fig.update_layout(
# 				height=600,
# 				width=800,
# 				title=dict(
# 					text="Top 0.1% Breakpoints - Degree Distribution",
# 					x=0.5,
# 					y=0.95,
# 					font=dict(size=18)
# 				),
# 				xaxis_title="Breakpoints",
# 				yaxis_title="Number of Patients",
# 				template="simple_white"
# 			)

# 		if bottom_bins:
# 			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
# 			bottom_degrees = [self.breakpoint_degrees[i] for i in bottom_bins]

# 			fig = go.Figure(
# 				data=go.Bar(
# 					x=bottom_breakpoints,
# 					y=bottom_degrees,
# 					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" for bp, deg in zip(bottom_breakpoints, bottom_degrees)],
# 					hoverinfo='text',
# 					marker_color='rgb(158,202,225)',
# 					marker_line_color='rgb(8,48,107)',
# 					marker_line_width=1.5,
# 					name="Bottom Breakpoints"
# 				)
# 			)

# 			fig.update_layout(
# 				height=600,
# 				width=800,
# 				title=dict(
# 					text="Bottom 0.1% Breakpoints - Degree Distribution",
# 					x=0.5,
# 					y=0.95,
# 					font=dict(size=18)
# 				),
# 				xaxis_title="Breakpoints",
# 				yaxis_title="Number of Patients",
# 				template="simple_white"
# 			)

# 		return fig
	
# 	def create_visualization(self) -> go.Figure:
# 		fig = make_subplots(
# 			rows=2, cols=2,
# 			subplot_titles=("Patient-Breakpoint Adjacency Matrix", 
# 							"Breakpoint Degree Distribution",
# 							"Patient Similarity Matrix", 
# 							"Breakpoint Co-occurrence Matrix"),
# 			specs=[[{"type": "heatmap"}, {"type": "bar"}],
# 					[{"type": "heatmap"}, {"type": "heatmap"}]]
# 		)

# 		# 1. Adjacency Matrix with custom hover text
# 		hover_text = [[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if self.adj_matrix[i][j] else 'No'}"
# 						for j, b in enumerate(self.breakpoints)]
# 						for i, p in enumerate(self.patients)]

# 		fig.add_trace(
# 			go.Heatmap(
# 				z=self.adj_matrix,
# 				x=self.breakpoints,
# 				y=self.patients,
# 				colorscale="Blues",
# 				showscale=True,
# 				hoverongaps=False,
# 				hoverinfo='text',
# 				text=hover_text,
# 				colorbar=dict(title="Connection"),
# 				name="Connections"
# 			),
# 			row=1, col=1
# 		)

# 		# 2. Degree Distribution with custom hover
# 		hover_text = [f"Breakpoint: {bp}<br>Connected to {deg} patients"
# 						for bp, deg in zip(self.breakpoints, self.breakpoint_degrees)]

# 		fig.add_trace(
# 			go.Bar(
# 				x=self.breakpoints,
# 				y=self.breakpoint_degrees,
# 				hovertext=hover_text,
# 				hoverinfo='text',
# 				marker_color='rgb(158,202,225)',
# 				marker_line_color='rgb(8,48,107)',
# 				marker_line_width=1.5,
# 				name="Breakpoint Degrees"
# 			),
# 			row=1, col=2
# 		)

# 		# 3. Patient Similarity Matrix
# 		fig.add_trace(
# 			go.Heatmap(
# 				z=1 - self.patient_similarity,
# 				x=self.patients,
# 				y=self.patients,
# 				colorscale="Viridis",
# 				showscale=True,
# 				colorbar=dict(title="Similarity"),
# 				name="Patient Similarity"
# 			),
# 			row=2, col=1
# 		)

# 		# 4. Breakpoint Co-occurrence
# 		fig.add_trace(
# 			go.Heatmap(
# 				z=1 - self.breakpoint_similarity,
# 				x=self.breakpoints,
# 				y=self.breakpoints,
# 				colorscale="Viridis",
# 				showscale=True,
# 				colorbar=dict(title="Co-occurrence"),
# 				name="Breakpoint Co-occurrence"
# 			),
# 			row=2, col=2
# 		)

# 		# Update layout with better styling
# 		fig.update_layout(
# 			height=1000,
# 			width=1200,
# 			title=dict(
# 				text="Network Analysis Dashboard",
# 				x=0.5,
# 				y=0.95,
# 				font=dict(size=24)
# 			),
# 			showlegend=False,
# 			template="simple_white"
# 		)

# 		# Update axes labels with better font sizes
# 		font_size = 14
# 		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=1)
# 		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=1, col=1)
# 		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=2)
# 		fig.update_yaxes(title_text="Number of Patients", title_font=dict(size=font_size), row=1, col=2)
# 		fig.update_xaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
# 		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
# 		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)
# 		fig.update_yaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)

# 		return fig

# 	def get_breakpoint_bins(self, top_percentile: float = 0.001, bottom_percentile: float = 0.001) -> tuple:
# 		"""
# 		Calculate the indexes of the breakpoints at the specified percentiles.
# 		Returns a tuple of two lists:
# 		- The first list contains the indexes of the top `top_percentile` breakpoints by degree.
# 		- The second list contains the indexes of the bottom `bottom_percentile` breakpoints by degree.
# 		"""
# 		sorted_degrees = sorted(self.breakpoint_degrees)
# 		top_cutoff = int(len(sorted_degrees) * top_percentile)
# 		bottom_cutoff = int(len(sorted_degrees) * (1 - bottom_percentile))

# 		top_bins = [i for i in range(top_cutoff)]
# 		bottom_bins = [i for i in range(bottom_cutoff, len(sorted_degrees))]

# 		return top_bins, bottom_bins

# 	def print_summary_stats(self):
# 		print(f"Network Summary Statistics:")
# 		print(f"---------------------------")
# 		print(f"Number of Patients: {len(self.patients)}")
# 		print(f"Number of Breakpoints: {len(self.breakpoints)}")
# 		print(f"Network Density: {self.density:.3f}")
# 		print(f"Average Patient Degree: {np.mean(self.patient_degrees):.2f}")
# 		print(f"Average Breakpoint Degree: {np.mean(self.breakpoint_degrees):.2f}")
# 		print(f"\nTop Breakpoints by Degree:")
# 		for bp, degree in sorted(zip(self.breakpoints, self.breakpoint_degrees), 
# 								key=lambda x: x[1], reverse=True)[:5]:
# 			print(f"  {bp}: {degree}")
# 		print(f"\nTop 10 Breakpoints by Degree Centrality:")
#     	# Create list of (breakpoint, centrality) tuples and sort by centrality
# 		centrality_pairs = list(zip(self.breakpoints, self.breakpoint_centrality))
# 		sorted_by_centrality = sorted(centrality_pairs, key=lambda x: x[1], reverse=True)
		
# 		# Print top 10
# 		for bp, centrality in sorted_by_centrality[:10]:
# 			print(f"  {bp}: {centrality:.3f}")
```


```python
import numpy as np
import networkx as nx
import plotly.graph_objects as go
from plotly.subplots import make_subplots
from scipy.spatial.distance import pdist, squareform
from networkx.algorithms.bipartite import density as bipartite_density
from scipy.sparse import csr_matrix

class NetworkAnalyzer:
	def __init__(self, df=None, patient_col=None, breakpoint_col=None, 
					precomputed_matrix=None, patients=None, breakpoints=None):
		"""
		Initialize NetworkAnalyzer with either DataFrame or precomputed matrix.
		
		Args:
			df (pl.DataFrame.Polars, optional): Input DataFrame HAS TO BE IN POLARS
			patient_col (str, optional): Column name for patients
			breakpoint_col (str, optional): Column name for breakpoints
			precomputed_matrix (csr_matrix, optional): Pre-computed sparse adjacency matrix
			patients (list, optional): List of patient IDs (required if using precomputed_matrix)
			breakpoints (list, optional): List of breakpoint IDs (required if using precomputed_matrix)
		"""
		if precomputed_matrix is not None:
			if patients is None or breakpoints is None:
				raise ValueError("Must provide patients and breakpoints lists with precomputed matrix")
			self.adj_matrix = precomputed_matrix.toarray()  # Convert to dense for existing functionality
			self.patients = patients
			self.breakpoints = breakpoints
		elif df is not None and patient_col and breakpoint_col:
			self.df = df
			self.patient_col = patient_col
			self.breakpoint_col = breakpoint_col
			
			# Get unique sets
			self.patients = sorted(df[patient_col].unique().to_list())
			self.breakpoints = sorted(df[breakpoint_col].unique().to_list())
			
			# Create adjacency matrix
			self.adj_matrix = self._create_adjacency_matrix()
		else:
			raise ValueError("Must provide either DataFrame with column names or precomputed matrix with labels")

		# Calculate network metrics
		self._calculate_metrics()

	def save_matrix(self, filename):
		"""
		Save the adjacency matrix in CSR format along with patient and breakpoint labels.
		
		Args:
			filename (str): Base filename to save the data (without extension)
		"""
		# Save the sparse matrix
		sparse_matrix = csr_matrix(self.adj_matrix)
		np.savez(f"{filename}_adjac_matrix.npz",
					data=sparse_matrix.data,
					indices=sparse_matrix.indices,
					indptr=sparse_matrix.indptr,
					shape=sparse_matrix.shape)
		
		# Save the labels
		np.save(f"{filename}_matrix_label_patients.npy", np.array(self.patients))
		np.save(f"{filename}_matrix_label_breakpoints.npy", np.array(self.breakpoints))

	@classmethod
	def load_from_files(cls, filename):
		"""
		Load a NetworkAnalyzer instance from saved files.
		
		Args:
			filename (str): Base filename (without extension) used when saving
			
		Returns:
			NetworkAnalyzer: New instance with loaded data
		"""
		# Load the sparse matrix
		loader = np.load(f"{filename}_adjac_matrix.npz")
		matrix = csr_matrix((loader['data'], loader['indices'], loader['indptr']),
							shape=loader['shape'])
		
		# Load the labels
		patients = np.load(f"{filename}_matrix_label_patients.npy").tolist()
		breakpoints = np.load(f"{filename}_matrix_label_breakpoints.npy").tolist()

		return cls(precomputed_matrix=matrix, patients=patients, breakpoints=breakpoints)

	def _create_adjacency_matrix(self) -> np.ndarray:
		"""Create the adjacency matrix from the input DataFrame."""
		matrix = np.zeros((len(self.patients), len(self.breakpoints)))
		connections = self.df.group_by(self.patient_col).agg(
			pl.col(self.breakpoint_col).alias('breakpoints')
		).to_dict(as_series=False)

		patient_idx = {p: i for i, p in enumerate(self.patients)}
		breakpoint_idx = {b: i for i, b in enumerate(self.breakpoints)}

		for i, patient in enumerate(connections[self.patient_col]):
			for bp in connections['breakpoints'][i]:
				matrix[patient_idx[patient]][breakpoint_idx[bp]] = 1

		return matrix

	def _calculate_metrics(self):
		"""Calculate various network metrics."""
		self.patient_degrees = np.sum(self.adj_matrix, axis=1)
		self.breakpoint_degrees = np.sum(self.adj_matrix, axis=0)
		self.patient_similarity = squareform(pdist(self.adj_matrix, metric='jaccard'))
		self.breakpoint_similarity = squareform(pdist(self.adj_matrix.T, metric='jaccard'))

		# Create bipartite graph manually
		G = nx.Graph()
		G.add_nodes_from(self.patients, bipartite=0)
		G.add_nodes_from(self.breakpoints, bipartite=1)
		for i, patient in enumerate(self.patients):
			for j, breakpoint in enumerate(self.breakpoints):
				if self.adj_matrix[i, j]:
					G.add_edge(patient, breakpoint)

		centrality = nx.degree_centrality(G)
		self.breakpoint_centrality = [centrality[bp] for bp in self.breakpoints]
		self.density = bipartite_density(G, self.breakpoints)

	def create_adjacency_matrix_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
		"""Create a standalone plot of the patient-breakpoint adjacency matrix."""
		if top_bins:
			top_breakpoints = [self.breakpoints[i] for i in top_bins]
			top_matrix = self.adj_matrix[:, top_bins]

			fig = go.Figure(
				data=go.Heatmap(
					z=top_matrix,
					x=top_breakpoints,
					y=self.patients,
					colorscale="Blues",
					showscale=True,
					hoverongaps=False,
					hoverinfo='text',
					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if top_matrix[i][j] else 'No'}"
							for j, b in enumerate(top_breakpoints)]
							for i, p in enumerate(self.patients)],
					colorbar=dict(title="Connection"),
					name="Top Breakpoints"
				)
			)

			fig.update_layout(
				height=800,
				width=1000,
				title=dict(
					text="Top 0.1% Breakpoints - Adjacency Matrix",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Patients",
				template="simple_white"
			)

		if bottom_bins:
			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
			bottom_matrix = self.adj_matrix[:, bottom_bins]

			fig = go.Figure(
				data=go.Heatmap(
					z=bottom_matrix,
					x=bottom_breakpoints,
					y=self.patients,
					colorscale="Blues",
					showscale=True,
					hoverongaps=False,
					hoverinfo='text',
					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if bottom_matrix[i][j] else 'No'}"
							for j, b in enumerate(bottom_breakpoints)]
							for i, p in enumerate(self.patients)],
					colorbar=dict(title="Connection"),
					name="Bottom Breakpoints"
				)
			)

			fig.update_layout(
				height=800,
				width=1000,
				title=dict(
					text="Bottom 0.1% Breakpoints - Adjacency Matrix",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Patients",
				template="simple_white"
			)

		return fig

	def create_degree_distribution_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
		"""Create a standalone plot of the breakpoint degree distribution."""
		if top_bins:
			top_breakpoints = [self.breakpoints[i] for i in top_bins]
			top_degrees = [self.breakpoint_degrees[i] for i in top_bins]

			fig = go.Figure(
				data=go.Bar(
					x=top_breakpoints,
					y=top_degrees,
					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" 
								for bp, deg in zip(top_breakpoints, top_degrees)],
					hoverinfo='text',
					marker_color='rgb(158,202,225)',
					marker_line_color='rgb(8,48,107)',
					marker_line_width=1.5,
					name="Top Breakpoints"
				)
			)

			fig.update_layout(
				height=600,
				width=800,
				title=dict(
					text="Top 0.1% Breakpoints - Degree Distribution",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Number of Patients",
				template="simple_white"
			)

		if bottom_bins:
			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
			bottom_degrees = [self.breakpoint_degrees[i] for i in bottom_bins]

			fig = go.Figure(
				data=go.Bar(
					x=bottom_breakpoints,
					y=bottom_degrees,
					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" 
								for bp, deg in zip(bottom_breakpoints, bottom_degrees)],
					hoverinfo='text',
					marker_color='rgb(158,202,225)',
					marker_line_color='rgb(8,48,107)',
					marker_line_width=1.5,
					name="Bottom Breakpoints"
				)
			)

			fig.update_layout(
				height=600,
				width=800,
				title=dict(
					text="Bottom 0.1% Breakpoints - Degree Distribution",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Number of Patients",
				template="simple_white"
			)

		return fig

	def create_visualization(self) -> go.Figure:
		"""Create a comprehensive visualization dashboard."""
		fig = make_subplots(
			rows=2, cols=2,
			subplot_titles=("Patient-Breakpoint Adjacency Matrix", 
							"Breakpoint Degree Distribution",
							"Patient Similarity Matrix", 
							"Breakpoint Co-occurrence Matrix"),
			specs=[[{"type": "heatmap"}, {"type": "bar"}],
					[{"type": "heatmap"}, {"type": "heatmap"}]]
		)

		# 1. Adjacency Matrix with custom hover text
		hover_text = [[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if self.adj_matrix[i][j] else 'No'}"
						for j, b in enumerate(self.breakpoints)]
						for i, p in enumerate(self.patients)]

		fig.add_trace(
			go.Heatmap(
				z=self.adj_matrix,
				x=self.breakpoints,
				y=self.patients,
				colorscale="Blues",
				showscale=True,
				hoverongaps=False,
				hoverinfo='text',
				text=hover_text,
				colorbar=dict(title="Connection"),
				name="Connections"
			),
			row=1, col=1
		)

		# 2. Degree Distribution with custom hover
		hover_text = [f"Breakpoint: {bp}<br>Connected to {deg} patients"
						for bp, deg in zip(self.breakpoints, self.breakpoint_degrees)]

		fig.add_trace(
			go.Bar(
				x=self.breakpoints,
				y=self.breakpoint_degrees,
				hovertext=hover_text,
				hoverinfo='text',
				marker_color='rgb(158,202,225)',
				marker_line_color='rgb(8,48,107)',
				marker_line_width=1.5,
				name="Breakpoint Degrees"
			),
			row=1, col=2
		)

		# 3. Patient Similarity Matrix
		fig.add_trace(
			go.Heatmap(
				z=1 - self.patient_similarity,
				x=self.patients,
				y=self.patients,
				colorscale="Viridis",
				showscale=True,
				colorbar=dict(title="Similarity"),
				name="Patient Similarity"
			),
			row=2, col=1
		)

		# 4. Breakpoint Co-occurrence
		fig.add_trace(
			go.Heatmap(
				z=1 - self.breakpoint_similarity,
				x=self.breakpoints,
				y=self.breakpoints,
				colorscale="Viridis",
				showscale=True,
				colorbar=dict(title="Co-occurrence"),
				name="Breakpoint Co-occurrence"
			),
			row=2, col=2
		)

		fig.update_layout(
			height=1000,
			width=1200,
			title=dict(
				text="Network Analysis Dashboard",
				x=0.5,
				y=0.95,
				font=dict(size=24)
			),
			showlegend=False,
			template="simple_white"
		)

		font_size = 14
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=1)
		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=1, col=1)
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=2)
		fig.update_yaxes(title_text="Number of Patients", title_font=dict(size=font_size), row=1, col=2)
		fig.update_xaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)
		fig.update_yaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)

		return fig
	
	def get_breakpoint_bins(self, top_percentile: float = 0.001, bottom_percentile: float = 0.001) -> tuple:
		"""
		Calculate the indexes of the breakpoints at the specified percentiles.
		Returns a tuple of two lists:
		- The first list contains the indexes of the top `top_percentile` breakpoints by degree.
		- The second list contains the indexes of the bottom `bottom_percentile` breakpoints by degree.
		"""
		sorted_degrees = sorted(self.breakpoint_degrees)
		top_cutoff = int(len(sorted_degrees) * top_percentile)
		bottom_cutoff = int(len(sorted_degrees) * (1 - bottom_percentile))

		top_bins = [i for i in range(top_cutoff)]
		bottom_bins = [i for i in range(bottom_cutoff, len(sorted_degrees))]

		return top_bins, bottom_bins

	def print_summary_stats(self):
		print(f"Network Summary Statistics:")
		print(f"---------------------------")
		print(f"Number of Patients: {len(self.patients)}")
		print(f"Number of Breakpoints: {len(self.breakpoints)}")
		print(f"Network Density: {self.density:.3f}")
		print(f"Average Patient Degree: {np.mean(self.patient_degrees):.2f}")
		print(f"Average Breakpoint Degree: {np.mean(self.breakpoint_degrees):.2f}")
		print(f"\nTop Breakpoints by Degree:")
		for bp, degree in sorted(zip(self.breakpoints, self.breakpoint_degrees), 
								key=lambda x: x[1], reverse=True)[:5]:
			print(f"  {bp}: {degree}")
		print(f"\nTop 10 Breakpoints by Degree Centrality:")
    	# Create list of (breakpoint, centrality) tuples and sort by centrality
		centrality_pairs = list(zip(self.breakpoints, self.breakpoint_centrality))
		sorted_by_centrality = sorted(centrality_pairs, key=lambda x: x[1], reverse=True)
		
		# Print top 10
		for bp, centrality in sorted_by_centrality[:10]:
			print(f"  {bp}: {centrality:.3f}")
```


```python
import numpy as np
import networkx as nx
import plotly.graph_objects as go
from plotly.subplots import make_subplots
from scipy.spatial.distance import pdist, squareform
from networkx.algorithms.bipartite import density as bipartite_density
from scipy.sparse import csr_matrix

class NetworkAnalyzer:
	def __init__(self, df=None, patient_col=None, breakpoint_col=None, 
					precomputed_matrix=None, patients=None, breakpoints=None):
		"""
		Initialize NetworkAnalyzer with either DataFrame or precomputed matrix.
		
		Args:
			df (pl.DataFrame.Polars, optional): Input DataFrame HAS TO BE IN POLARS
			patient_col (str, optional): Column name for patients
			breakpoint_col (str, optional): Column name for breakpoints
			precomputed_matrix (csr_matrix, optional): Pre-computed sparse adjacency matrix
			patients (list, optional): List of patient IDs (required if using precomputed_matrix)
			breakpoints (list, optional): List of breakpoint IDs (required if using precomputed_matrix)
		"""
		if precomputed_matrix is not None:
			if patients is None or breakpoints is None:
				raise ValueError("Must provide patients and breakpoints lists with precomputed matrix")
			# Keep matrix in sparse format
			self.adj_matrix_sparse = precomputed_matrix
			self.patients = patients
			self.breakpoints = breakpoints
		elif df is not None and patient_col and breakpoint_col:
			self.df = df
			self.patient_col = patient_col
			self.breakpoint_col = breakpoint_col
			
			# Get unique sets
			self.patients = sorted(df[patient_col].unique().to_list())
			self.breakpoints = sorted(df[breakpoint_col].unique().to_list())
			
			# Create sparse adjacency matrix
			self.adj_matrix_sparse = self._create_adjacency_matrix()
		else:
			raise ValueError("Must provide either DataFrame with column names or precomputed matrix with labels")

		# Don't calculate metrics immediately - do it lazily
		self._metrics_calculated = False
		
	def _ensure_metrics_calculated(self):
		"""Calculate metrics if they haven't been calculated yet."""
		if not self._metrics_calculated:
			self._calculate_metrics()
			self._metrics_calculated = True

	def _create_adjacency_matrix(self) -> csr_matrix:
		"""Create the sparse adjacency matrix from the input DataFrame."""
		matrix = np.zeros((len(self.patients), len(self.breakpoints)))
		connections = self.df.group_by(self.patient_col).agg(
			pl.col(self.breakpoint_col).alias('breakpoints')
		).to_dict(as_series=False)

		patient_idx = {p: i for i, p in enumerate(self.patients)}
		breakpoint_idx = {b: i for i, b in enumerate(self.breakpoints)}

		for i, patient in enumerate(connections[self.patient_col]):
			for bp in connections['breakpoints'][i]:
				matrix[patient_idx[patient]][breakpoint_idx[bp]] = 1

		return csr_matrix(matrix)

	def _calculate_metrics(self):
		"""Calculate various network metrics."""
		# Convert to dense only when needed for specific calculations
		dense_matrix = self.adj_matrix_sparse.toarray()
		
		self.patient_degrees = np.asarray(self.adj_matrix_sparse.sum(axis=1)).flatten()
		self.breakpoint_degrees = np.asarray(self.adj_matrix_sparse.sum(axis=0)).flatten()
		
		# Only calculate similarity matrices if needed for visualization
		self.patient_similarity = squareform(pdist(dense_matrix, metric='jaccard'))
		self.breakpoint_similarity = squareform(pdist(dense_matrix.T, metric='jaccard'))

		# Create bipartite graph more efficiently
		G = nx.Graph()
		G.add_nodes_from(range(len(self.patients)), bipartite=0)
		G.add_nodes_from(range(len(self.patients), len(self.patients) + len(self.breakpoints)), bipartite=1)
		
		# Add edges using sparse matrix coordinates
		rows, cols = self.adj_matrix_sparse.nonzero()
		edges = zip(rows, cols + len(self.patients))
		G.add_edges_from(edges)

		self.density = bipartite_density(G, range(len(self.patients), len(self.patients) + len(self.breakpoints)))
		
		# Calculate centrality
		centrality = nx.degree_centrality(G)
		self.breakpoint_centrality = [centrality[i + len(self.patients)] for i in range(len(self.breakpoints))]

	@classmethod
	def load_from_files(cls, filename):
		"""
		Load a NetworkAnalyzer instance from saved files.
		
		Args:
			filename (str): Base filename (without extension) used when saving
			
		Returns:
			NetworkAnalyzer: New instance with loaded data
		"""
		# Load the sparse matrix
		loader = np.load(f"{filename}_adjac_matrix.npz")
		matrix = csr_matrix((loader['data'], loader['indices'], loader['indptr']),
							shape=loader['shape'])
		
		# Load the labels
		patients = np.load(f"{filename}_matrix_label_patients.npy").tolist()
		breakpoints = np.load(f"{filename}_matrix_label_breakpoints.npy").tolist()

		return cls(precomputed_matrix=matrix, patients=patients, breakpoints=breakpoints)
	
	def create_adjacency_matrix_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
		"""Create a standalone plot of the patient-breakpoint adjacency matrix."""
		# Ensure metrics are calculated before accessing them
		self._ensure_metrics_calculated()
		# Convert to dense only when needed for specific calculations
		dense_matrix = self.adj_matrix_sparse.toarray()

		if top_bins:
			top_breakpoints = [self.breakpoints[i] for i in top_bins]
			top_matrix = dense_matrix[:, top_bins]

			fig = go.Figure(
				data=go.Heatmap(
					z=top_matrix,
					x=top_breakpoints,
					y=self.patients,
					colorscale="Blues",
					showscale=True,
					hoverongaps=False,
					hoverinfo='text',
					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if top_matrix[i][j] else 'No'}"
							for j, b in enumerate(top_breakpoints)]
							for i, p in enumerate(self.patients)],
					colorbar=dict(title="Connection"),
					name="Top Breakpoints"
				)
			)

			fig.update_layout(
				height=800,
				width=1000,
				title=dict(
					text="Top 0.1% Breakpoints - Adjacency Matrix",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Patients",
				template="simple_white"
			)

		if bottom_bins:
			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
			bottom_matrix = dense_matrix[:, bottom_bins]

			fig = go.Figure(
				data=go.Heatmap(
					z=bottom_matrix,
					x=bottom_breakpoints,
					y=self.patients,
					colorscale="Blues",
					showscale=True,
					hoverongaps=False,
					hoverinfo='text',
					text=[[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if bottom_matrix[i][j] else 'No'}"
							for j, b in enumerate(bottom_breakpoints)]
							for i, p in enumerate(self.patients)],
					colorbar=dict(title="Connection"),
					name="Bottom Breakpoints"
				)
			)

			fig.update_layout(
				height=800,
				width=1000,
				title=dict(
					text="Bottom 0.1% Breakpoints - Adjacency Matrix",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Patients",
				template="simple_white"
			)

		return fig

	def create_degree_distribution_plot(self, top_bins: list = None, bottom_bins: list = None) -> go.Figure:
		"""Create a standalone plot of the breakpoint degree distribution."""
		# Ensure metrics are calculated before accessing them
		self._ensure_metrics_calculated()
		if top_bins:
			top_breakpoints = [self.breakpoints[i] for i in top_bins]
			top_degrees = [self.breakpoint_degrees[i] for i in top_bins]

			fig = go.Figure(
				data=go.Bar(
					x=top_breakpoints,
					y=top_degrees,
					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" 
								for bp, deg in zip(top_breakpoints, top_degrees)],
					hoverinfo='text',
					marker_color='rgb(158,202,225)',
					marker_line_color='rgb(8,48,107)',
					marker_line_width=1.5,
					name="Top Breakpoints"
				)
			)

			fig.update_layout(
				height=600,
				width=800,
				title=dict(
					text="Top 0.1% Breakpoints - Degree Distribution",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Number of Patients",
				template="simple_white"
			)

		if bottom_bins:
			bottom_breakpoints = [self.breakpoints[i] for i in bottom_bins]
			bottom_degrees = [self.breakpoint_degrees[i] for i in bottom_bins]

			fig = go.Figure(
				data=go.Bar(
					x=bottom_breakpoints,
					y=bottom_degrees,
					hovertext=[f"Breakpoint: {bp}<br>Connected to {deg} patients" 
								for bp, deg in zip(bottom_breakpoints, bottom_degrees)],
					hoverinfo='text',
					marker_color='rgb(158,202,225)',
					marker_line_color='rgb(8,48,107)',
					marker_line_width=1.5,
					name="Bottom Breakpoints"
				)
			)

			fig.update_layout(
				height=600,
				width=800,
				title=dict(
					text="Bottom 0.1% Breakpoints - Degree Distribution",
					x=0.5,
					y=0.95,
					font=dict(size=18)
				),
				xaxis_title="Breakpoints",
				yaxis_title="Number of Patients",
				template="simple_white"
			)

		return fig

	def create_visualization(self) -> go.Figure:
		"""Create a comprehensive visualization dashboard."""
		# Ensure metrics are calculated before accessing them
		self._ensure_metrics_calculated()
		# Convert to dense only when needed for specific calculations
		dense_matrix = self.adj_matrix_sparse.toarray()
		fig = make_subplots(
			rows=2, cols=2,
			subplot_titles=("Patient-Breakpoint Adjacency Matrix", 
							"Breakpoint Degree Distribution",
							"Patient Similarity Matrix", 
							"Breakpoint Co-occurrence Matrix"),
			specs=[[{"type": "heatmap"}, {"type": "bar"}],
					[{"type": "heatmap"}, {"type": "heatmap"}]]
		)

		# 1. Adjacency Matrix with custom hover text
		hover_text = [[f"Patient: {p}<br>Breakpoint: {b}<br>Connected: {'Yes' if dense_matrix[i][j] else 'No'}"
						for j, b in enumerate(self.breakpoints)]
						for i, p in enumerate(self.patients)]

		fig.add_trace(
			go.Heatmap(
				z=dense_matrix,
				x=self.breakpoints,
				y=self.patients,
				colorscale="Blues",
				showscale=True,
				hoverongaps=False,
				hoverinfo='text',
				text=hover_text,
				colorbar=dict(title="Connection"),
				name="Connections"
			),
			row=1, col=1
		)

		# 2. Degree Distribution with custom hover
		hover_text = [f"Breakpoint: {bp}<br>Connected to {deg} patients"
						for bp, deg in zip(self.breakpoints, self.breakpoint_degrees)]

		fig.add_trace(
			go.Bar(
				x=self.breakpoints,
				y=self.breakpoint_degrees,
				hovertext=hover_text,
				hoverinfo='text',
				marker_color='rgb(158,202,225)',
				marker_line_color='rgb(8,48,107)',
				marker_line_width=1.5,
				name="Breakpoint Degrees"
			),
			row=1, col=2
		)

		# 3. Patient Similarity Matrix
		fig.add_trace(
			go.Heatmap(
				z=1 - self.patient_similarity,
				x=self.patients,
				y=self.patients,
				colorscale="Viridis",
				showscale=True,
				colorbar=dict(title="Similarity"),
				name="Patient Similarity"
			),
			row=2, col=1
		)

		# 4. Breakpoint Co-occurrence
		fig.add_trace(
			go.Heatmap(
				z=1 - self.breakpoint_similarity,
				x=self.breakpoints,
				y=self.breakpoints,
				colorscale="Viridis",
				showscale=True,
				colorbar=dict(title="Co-occurrence"),
				name="Breakpoint Co-occurrence"
			),
			row=2, col=2
		)

		fig.update_layout(
			height=1000,
			width=1200,
			title=dict(
				text="Network Analysis Dashboard",
				x=0.5,
				y=0.95,
				font=dict(size=24)
			),
			showlegend=False,
			template="simple_white"
		)

		font_size = 14
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=1)
		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=1, col=1)
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=1, col=2)
		fig.update_yaxes(title_text="Number of Patients", title_font=dict(size=font_size), row=1, col=2)
		fig.update_xaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
		fig.update_yaxes(title_text="Patients", title_font=dict(size=font_size), row=2, col=1)
		fig.update_xaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)
		fig.update_yaxes(title_text="Breakpoints", title_font=dict(size=font_size), row=2, col=2)

		return fig
	
	def get_breakpoint_bins(self, top_percentile: float = 0.001, bottom_percentile: float = 0.001) -> tuple:
		"""
		Calculate the indexes of the breakpoints at the specified percentiles.
		Returns a tuple of two lists:
		- The first list contains the indexes of the top `top_percentile` breakpoints by degree.
		- The second list contains the indexes of the bottom `bottom_percentile` breakpoints by degree.
		"""
		# Ensure metrics are calculated before accessing them
		self._ensure_metrics_calculated()
		sorted_degrees = sorted(self.breakpoint_degrees)
		top_cutoff = int(len(sorted_degrees) * top_percentile)
		bottom_cutoff = int(len(sorted_degrees) * (1 - bottom_percentile))

		top_bins = [i for i in range(top_cutoff)]
		bottom_bins = [i for i in range(bottom_cutoff, len(sorted_degrees))]

		return top_bins, bottom_bins

	def print_summary_stats(self):
		# Ensure metrics are calculated before accessing them
		self._ensure_metrics_calculated()
		print(f"Network Summary Statistics:")
		print(f"---------------------------")
		print(f"Number of Patients: {len(self.patients)}")
		print(f"Number of Breakpoints: {len(self.breakpoints)}")
		print(f"Network Density: {self.density:.3f}")
		print(f"Average Patient Degree: {np.mean(self.patient_degrees):.2f}")
		print(f"Average Breakpoint Degree: {np.mean(self.breakpoint_degrees):.2f}")
		print(f"\nTop Breakpoints by Degree:")
		for bp, degree in sorted(zip(self.breakpoints, self.breakpoint_degrees), 
								key=lambda x: x[1], reverse=True)[:5]:
			print(f"  {bp}: {degree}")
		print(f"\nTop 10 Breakpoints by Degree Centrality:")
    	# Create list of (breakpoint, centrality) tuples and sort by centrality
		centrality_pairs = list(zip(self.breakpoints, self.breakpoint_centrality))
		sorted_by_centrality = sorted(centrality_pairs, key=lambda x: x[1], reverse=True)
		
		# Print top 10
		for bp, centrality in sorted_by_centrality[:10]:
			print(f"  {bp}: {centrality:.3f}")
```


```python
# create toy data
np.random.seed(42)  # for reproducibility

patients = [f'P{i}' for i in range(1, 21)]  # 20 patients
breakpoints = [f'BP{i}' for i in range(1, 16)]  # 15 breakpoints

# Create random connections (each patient has 2-6 breakpoints)
data = []
for patient in patients:
	num_breakpoints = np.random.randint(2, 7)
	patient_breakpoints = np.random.choice(breakpoints, size=num_breakpoints, replace=False)
	for bp in patient_breakpoints:
		data.append({'patient_id': patient, 'breakpoint_id': bp})

# Create Polars DataFrame
df = pl.DataFrame(data)

# now test the class
# Create and display visualization
analyzer = NetworkAnalyzer(df, patient_col='patient_id', breakpoint_col='breakpoint_id')
fig = analyzer.create_visualization()
fig.show()

# Print summary statistics
analyzer.print_summary_stats()
```

Now run our data to create an instance of the class.


```python
# analyzer_my = NetworkAnalyzer(bp_sample_array, patient_col='sampleID', breakpoint_col='breakpointID')
```


```python
# bpsample_poldf = pl.from_pandas(bpsample_pdf_unique)
# analyzer_my_v2 = NetworkAnalyzer(bpsample_poldf, patient_col='sampleID', breakpoint_col='breakpointID')
```


```python
# Print summary statistics
# analyzer_my_v2.print_summary_stats()
```

NOTE: Use the script below to save the Analyzer.adj_matrix to file.


```python
# from scipy.sparse import csr_matrix

# def save_sparse_csr(filename, array):
#     # Convert to CSR matrix if it's not already one
#     if not isinstance(array, csr_matrix):
#         array = csr_matrix(array)
#     # note that .npz extension is added automatically
#     np.savez(filename, data=array.data, indices=array.indices,
#              indptr=array.indptr, shape=array.shape)

# def load_sparse_csr(filename):
#     # here we need to add .npz extension manually
#     loader = np.load(filename + '.npz')
#     return csr_matrix((loader['data'], loader['indices'], loader['indptr']),
#                       shape=loader['shape'])

# save_sparse_csr('output/adj_matrix_mybrca', analyzer_my.adj_matrix)
```


```python
# with the NetworkAnalyzer v2 class, we can call the method save_matrix directly

# analyzer_my_v2.save_matrix('output/adj_matrix_mybrca_v2')
```


```python
# try reloading into an instance using the decorated class method load_from_file

# analyzer_my_v2_reloaded = NetworkAnalyzer.load_from_files('output/adj_matrix_mybrca_v2')
```

The distribution of the Sharedness Degree of each unique breakpoint, is as expected, skewed towards having a lot of unique, patient-specific connections, and very few shared breakpoints across patients. We can try to visualize the adjacency matrix, but because of the massive matrix dimension we have (988 patients x 43927 unique breakpoints, it is best to bin the matrix and plot only the top 0.1% breakpoints having the highest sharedness degree. middle 0.1%, and bottom 0.1%.


```python
# loaded_matx = load_sparse_csr('output/adj_matrix_mybrca')
# loaded_matx = loaded_matx.toarray()
# loaded_matx
```


```python
# # Print summary statistics
# analyzer_my.print_summary_stats()

# # Get the top and bottom bin indexes (0.1% each)
# top_bins, bottom_bins = analyzer_my.get_breakpoint_bins(top_percentile=0.001, bottom_percentile=0.001)

# # Create the adjacency matrix plot with top and bottom breakpoints
# top_adjacency_plot = analyzer_my.create_adjacency_matrix_plot(top_bins=top_bins)
# top_adjacency_plot.show()
# bottom_adjacency_plot = analyzer_my.create_adjacency_matrix_plot(bottom_bins=bottom_bins)
# bottom_adjacency_plot.show()

```


```python
# # Create the degree distribution plot with top and bottom breakpoints
# top_degree_plot = analyzer_my.create_degree_distribution_plot(top_bins=top_bins)
# top_degree_plot.show()
# bottom_degree_plot = analyzer_my.create_degree_distribution_plot(bottom_bins=bottom_bins)
# bottom_degree_plot.show()
```

### Filter Out non-TNBCs

We can filter out rows corresponding to `sampleID` more than 172, because these are not TNBC samples.




```python
# import matplotlib.pyplot as plt

# # Count the number of unique breakpointIDs for each unique_samples value
# sharedness_counts = (
#     norm_filtered_summary_df
#     .group_by('unique_samples')
#     .agg(
#         pl.col('breakpointID').n_unique()
#         .alias('unique_bp_count')
#     )
#     .sort('unique_samples')
# )

# # Prepare the data for plotting
# x = sharedness_counts['unique_samples'].to_list()
# y = sharedness_counts['unique_bp_count'].to_list()

# # Create the bar plot
# plt.figure(figsize=(10, 8), dpi=300)
# plt.bar(x, y)

# # Set labels and title
# plt.xlabel('Sharedness Score (Number of Patients A Unique FT Is Observed)')
# plt.ylabel('Count of Unique FTs')
# plt.title('Frequency of Unique Tumor-Specific FTs by Sharedness Score')

# # Rotate x-axis labels for better readability
# plt.xticks(rotation=90)

# plt.show()
```


```python
# # Count the number of unique breakpointIDs for each unique_samples value
# sharedness_score = (
#     norm_filtered_summary_tnbc_df
#     .group_by('unique_samples')
#     .agg(
#         pl.col('breakpointID').n_unique()
#         .alias('unique_ft_count')
#     )
#     .sort('unique_samples')
# )
# sharedness_score
```


```python
# sharedness_score = sharedness_score.with_columns([
#         pl.col('unique_samples').cast(pl.Utf8)
#     ])
# sharedness_score
```


```python
# import matplotlib.pyplot as plt
# import seaborn as sns
# import polars as pl

# # Create the bar plot
# plt.figure(figsize=(10, 8), dpi=300)
# sns.barplot(x=sharedness_score['unique_samples'],y=sharedness_score['unique_ft_count'], color='steelblue')

# # Add value labels on top of the bars
# for i, v in enumerate(sharedness_score['unique_ft_count']):
#     plt.text(i, v, str(v), color='black', ha='center', fontweight='bold', fontsize=8)
    

# # Set labels and title
# plt.xlabel('Sharedness Score (Number of Patients A Unique FT Is Observed)')
# plt.ylabel('Count of Unique FTs')
# plt.title('Frequency of Unique Tumor-Specific FTs by Sharedness Score')

# # Rotate x-axis labels for better readability
# plt.xticks(rotation=90)

# plt.show()
```
