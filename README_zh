# Chinese
# 文章分配程序文档

## 中文文档

### 简介
此程序用于将提交的文章分配给审稿人。它根据审稿人的偏好和每篇文章的主题来进行分配，确保每篇文章都能被至少一位审稿人审阅。

### 输入文件
- `BioCAS 2023 Finalized RCM List 25Jun23.xlsx`：包含审稿人的偏好和回复状态。
- `db_extract_BIOCAS_2023.xlsx`：包含所有提交的文章信息。

### 输出文件
- `distribution_update_8.xlsx`：包含分配结果和未分配的文章。

### 使用方法
1. 将审稿人偏好表和文章提交表放在指定路径下。
2. 运行程序，生成分配结果文件。

### 代码详解
```python
import pandas as pd
import numpy as np
import networkx as nx

# 定义审稿人偏好和所有提交的文件路径
RCM_sheet = './2023/BioCAS 2023 Finalized RCM List 25Jun23.xlsx'
ALL_SUBMISSIONS = './2023/db_extract_BIOCAS_2023.xlsx'

# 读取审稿人偏好数据
preference = pd.read_excel(RCM_sheet, sheet_name='RCM & Tracks', header=1)
# 过滤出仅接受审稿的审稿人
preference = preference[preference["Reply (1: accept; blank: wait for reply)"] == 1]
# 将所有NaN值填充为0
preference = preference.fillna(0)

# 将偏好列转换为整数类型
tracks = preference.columns[3:-5]
dtype = ['int'] * len(tracks)
convert_dict = dict(zip(tracks, dtype))
preference = preference.astype(convert_dict)

# 按审稿人感兴趣的轨道数量进行排序（升序）
original_index = preference.index
columns_to_sort = preference.columns[3:-5]
column_counts = preference[columns_to_sort].sum(axis=1)
preference = preference.iloc[column_counts.sort_values(ascending=True).index]

# 提取审稿人的邮件列表
reviewers_maillist = preference[['Full Name', 'Email']]

# 读取稿件数据
manuscripts = pd.read_excel(ALL_SUBMISSIONS, dtype={'Paper ID': str, 'Track ID': int}, header=1)
# 过滤出Track ID小于17的稿件
manuscripts_1_16 = manuscripts[manuscripts['Track ID'] < 17]

# 确保数据一致性
assert len(set(manuscripts_1_16['Track Name'])) == 16
assert len(tracks) == 16
assert set(tracks) - set(manuscripts_1_16['Track Name']) == set()

# 为网络流输入生成审稿人偏好字符串
reviewers = ""
for idx in range(len(preference)):
    if not len(preference.iloc[idx][preference.iloc[idx].notna()].index[3:-1]):
        print(f"Reviewer: {preference.iloc[idx]['First Name'] + ',' + preference.iloc[idx]['Last Name']} without any preference, idx={idx}.")
        continue
    reviewer_idx = preference.iloc[idx]['First Name'] + ", " + preference.iloc[idx]['Last Name'] + ":" + '|'.join([item for item in preference.iloc[idx][preference.iloc[idx] != 0].index[3:-2]]) + ";"
    reviewers += reviewer_idx

# 为网络流输入生成稿件字符串
essay_track_i_string = ""
for i in range(len(manuscripts_1_16)):
    essay_track_i = f"{manuscripts_1_16.iloc[i]['Paper ID']}:{manuscripts_1_16.iloc[i]['Track Name']}" + ";"
    essay_track_i_string += essay_track_i

essays = essay_track_i_string

# 将审稿人和稿件字符串分割成列表
reviewers = reviewers.split(';')
essays = essays.split(';')

# 设置每个审稿人和每篇文章的最大审阅数量
max_reviews_per_reviewer = 7
max_reviews_per_essay = 1

# 创建用于网络流算法的有向图
G = nx.DiGraph()

# 存储每篇文章主题的字典
D = {}  
for E in essays:
    try:
        essay, subjects = E.split(':')
    except ValueError:
        print(E)
    G.add_edge('source', essay, capacity=max_reviews_per_essay, weight=0)
    D[essay] = set(subjects.split('|'))

# 特殊审稿人最大审阅数量字典
special_max_review_reviewers = {}

# 检查审稿人是否为文章作者的函数
def self_reviewers(essay, reviewer):
    reviewer = "".join(reviewer.split(','))
    if reviewer in manuscripts_1_16[manuscripts_1_16['Paper ID'] == essay]['Author Full Names'].values[0]:
        return True
    return False

# 根据审稿人偏好和文章主题添加边到图中
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

# 计算最小费用最大流
mincostFlow = nx.max_flow_min_cost(G, 'source', 'sink')

# 创建每个审稿人分配的文章映射
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

# 创建审稿人邮件列表
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

# 创建审稿人及其偏好轨道的映射
reviewer_track_mapping = {}
for i in range(len(preference)):
    tracks_i = []
    for j, item in enumerate(preference[preference.columns[3:-3]].iloc[i].tolist()):
        if item:
            tracks_i.append(j + 1)
    reviewer_track_mapping[f"{preference.iloc[i]['First Name']}, {preference.iloc[i]['Last Name']}"] = tracks_i

# 创建包含分配结果的数据框
essay_reviewer_df = pd.DataFrame(data={
    'Reviewer': essay_reviewer_mapping.keys(),
    'Reviewers track': reviewer_track_mapping.values(),
    'EssayList': essay_reviewer_mapping.values(),
    'Mail': MailLst
})

# 找出未分配给任何审稿人的文章
no_assigned_set = set(manuscripts_1_16['Paper ID']) - set(essay_assigned_total)

# 创建包含未分配文章的数据框
EssayNoReviewer = pd.DataFrame(data={
    'EssayNoReviewer': list(set(manuscripts_1_16['Paper ID']) - set(essay_assigned_total)),
    'track': manuscripts_1_16[manuscripts_1_16['Paper ID'].isin(no_assigned_set)]['Track ID'].astype(int).tolist()
})

# 根据原始索引重新排序数据框
essay_reviewer_df = essay_reviewer_df.reindex(original_index)

# 将分配结果写入Excel文件
with pd.ExcelWriter('./2023/distribution_update_8.xlsx') as writer:
    essay_reviewer_df.to_excel(writer, sheet_name='Assigned')
    EssayNoReviewer.to_excel(writer, sheet_name='Not assigned')
