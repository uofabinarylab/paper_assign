# English
# Article Assignment Program Documentation

## Introduction
This program is designed to assign submitted articles to reviewers based on their preferences and the topics of the articles. The goal is to ensure that each article is reviewed by at least one reviewer.

## Input Files
- `BioCAS 2023 Finalized RCM List 25Jun23.xlsx`: Contains the preferences and response status of the reviewers.
- `db_extract_BIOCAS_2023.xlsx`: Contains information about all submitted articles.

## Output Files
- `distribution_update_8.xlsx`: Contains the assignment results and the list of unassigned articles.

## Usage
1. Place the reviewer preference list and article submission list in the specified paths.
2. Run the program to generate the assignment results file.

## Code Explanation
```python
import pandas as pd
import numpy as np
import networkx as nx

# Define paths for reviewer preferences and all submissions files
RCM_sheet = './2023/BioCAS 2023 Finalized RCM List 25Jun23.xlsx'
ALL_SUBMISSIONS = './2023/db_extract_BIOCAS_2023.xlsx'

# Read reviewer preference data
preference = pd.read_excel(RCM_sheet, sheet_name='RCM & Tracks', header=1)
# Filter out reviewers who accepted to review
preference = preference[preference["Reply (1: accept; blank: wait for reply)"] == 1]
# Fill all NaN values with 0
preference = preference.fillna(0)

# Convert preference columns to integer type
tracks = preference.columns[3:-5]
dtype = ['int'] * len(tracks)
convert_dict = dict(zip(tracks, dtype))
preference = preference.astype(convert_dict)

# Sort reviewers by the number of tracks they are interested in (ascending order)
original_index = preference.index
columns_to_sort = preference.columns[3:-5]
column_counts = preference[columns_to_sort].sum(axis=1)
preference = preference.iloc[column_counts.sort_values(ascending=True).index]

# Extract reviewers' email list
reviewers_maillist = preference[['Full Name', 'Email']]

# Read manuscript data
manuscripts = pd.read_excel(ALL_SUBMISSIONS, dtype={'Paper ID': str, 'Track ID': int}, header=1)
# Filter manuscripts with Track ID less than 17
manuscripts_1_16 = manuscripts[manuscripts['Track ID'] < 17]

# Ensure data consistency
assert len(set(manuscripts_1_16['Track Name'])) == 16
assert len(tracks) == 16
assert set(tracks) - set(manuscripts_1_16['Track Name']) == set()

# Generate reviewer preferences string for network flow input
reviewers = ""
for idx in range(len(preference)):
    if not len(preference.iloc[idx][preference.iloc[idx].notna()].index[3:-1]):
        print(f"Reviewer: {preference.iloc[idx]['First Name'] + ',' + preference.iloc[idx]['Last Name']} without any preference, idx={idx}.")
        continue
    reviewer_idx = preference.iloc[idx]['First Name'] + ", " + preference.iloc[idx]['Last Name'] + ":" + '|'.join([item for item in preference.iloc[idx][preference.iloc[idx] != 0].index[3:-2]]) + ";"
    reviewers += reviewer_idx

# Generate manuscript strings for network flow input
essay_track_i_string = ""
for i in range(len(manuscripts_1_16)):
    essay_track_i = f"{manuscripts_1_16.iloc[i]['Paper ID']}:{manuscripts_1_16.iloc[i]['Track Name']}" + ";"
    essay_track_i_string += essay_track_i

essays = essay_track_i_string

# Split reviewer and manuscript strings into lists
reviewers = reviewers.split(';')
essays = essays.split(';')

# Set the maximum number of reviews per reviewer and per essay
max_reviews_per_reviewer = 7
max_reviews_per_essay = 1

# Create a directed graph for the network flow algorithm
G = nx.DiGraph()

# Dictionary to store the subjects for each essay
D = {}
for E in essays:
    try:
        essay, subjects = E.split(':')
    except ValueError:
        print(E)
    G.add_edge('source', essay, capacity=max_reviews_per_essay, weight=0)
    D[essay] = set(subjects.split('|'))

# Dictionary for special reviewers' maximum review counts
special_max_review_reviewers = {}

# Function to check if a reviewer is an author of an essay
def self_reviewers(essay, reviewer):
    reviewer = "".join(reviewer.split(','))
    if reviewer in manuscripts_1_16[manuscripts_1_16['Paper ID'] == essay]['Author Full Names'].values[0]:
        return True
    return False

# Add edges to the graph based on reviewer preferences and essay topics
for R in reviewers:
    try:
        reviewer, subjects = R.split(':')
    except ValueError:
        print(R)
    if reviewer in special_max_review_reviewers:
        print(f"Find special reviewer: {reviewer}")
        G.add_edge(reviewer, 'sink', capacity=special_max_review_reviewers[reviewer], weight=0)
    else:
        G.add_edge(reviewer, 'sink', capacity=max_reviews_per_reviewer, weight=0)
    reviewer_subjects = set(subjects.split('|'))
    for essay, essay_subjects in D.items():
        if (essay_subjects & reviewer_subjects) and not self_reviewers(essay, reviewer):
            G.add_edge(essay, reviewer, capacity=1, weight=0)
        else:
            G.add_edge(essay, reviewer, capacity=1, weight=100)

# Compute the minimum cost maximum flow
mincostFlow = nx.max_flow_min_cost(G, 'source', 'sink')

# Create mapping of essays assigned to each reviewer
essay_reviewer_mapping = {}
essay_assigned_total = []

for R in reviewers:
    essay_assigned = []
    try:
        reviewer, subjects = R.split(':')
    except ValueError:
        print(R)
    for essay in D:
        if mincostFlow[essay][reviewer] > 0.5:
            essay_assigned.append(f"{essay}_t{manuscripts_1_16[manuscripts_1_16['Paper ID'] == essay]['Track ID'].values[0]}")
            essay_assigned_total.append(essay)
    essay_reviewer_mapping[reviewer] = essay_assigned

# Create list of reviewers' emails
MailLst = []
for key in essay_reviewer_mapping:
    FirstName = key.split(',')[0].strip().replace('ó', 'o')
    LastName = key.split(',')[1].strip().replace('ó', 'o')
    try:
        MailLst.append(reviewers_maillist[reviewers_maillist['Full Name'] == f"{FirstName} {LastName}"].Email.values[0].strip())
    except:
        print(reviewers_maillist[reviewers_maillist['Full Name'] == f"{FirstName} {LastName}"])
        print(f"Cannot found: {FirstName} {LastName}")
        continue

# Create mapping of reviewers and their preferred tracks
reviewer_track_mapping = {}
for i in range(len(preference)):
    tracks_i = []
    for j, item in enumerate(preference[preference.columns[3:-3]].iloc[i].tolist()):
        if item:
            tracks_i.append(j + 1)
    reviewer_track_mapping[f"{preference.iloc[i]['First Name']}, {preference.iloc[i]['Last Name']}"] = tracks_i

# Create a DataFrame with assignment results
essay_reviewer_df = pd.DataFrame(data={
    'Reviewer': essay_reviewer_mapping.keys(),
    'Reviewers track': reviewer_track_mapping.values(),
    'EssayList': essay_reviewer_mapping.values(),
    'Mail': MailLst
})

# Find essays that were not assigned to any reviewer
no_assigned_set = set(manuscripts_1_16['Paper ID']) - set(essay_assigned_total)

# Create a DataFrame for unassigned essays
EssayNoReviewer = pd.DataFrame(data={
    'EssayNoReviewer': list(set(manuscripts_1_16['Paper ID']) - set(essay_assigned_total)),
    'track': manuscripts_1_16[manuscripts_1_16['Paper ID'].isin(no_assigned_set)]['Track ID'].astype(int).tolist()
})

# Reindex the DataFrame based on the original index
essay_reviewer_df = essay_reviewer_df.reindex(original_index)

# Write assignment results to an Excel file
with pd.ExcelWriter('./2023/distribution_update_8.xlsx') as writer:
    essay_reviewer_df.to_excel(writer, sheet_name='Assigned')
    EssayNoReviewer.to_excel(writer, sheet_name='Not assigned')



