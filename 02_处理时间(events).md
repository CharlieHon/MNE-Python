#mne #eeg #python 

本教程介绍了事件表示以及如何使用事件数组选择子数据。

首先，导入必要的模块，加载一些历程数据并在数据读入RAM前将其剪切为60s的长度来节省内存。

```python
import os
import numpy as np
import mne

sample_data_folder = mne.datasets.sample.data_path()
sample_data_raw_file = os.path.join(sample_data_folder, 'MEG', 'sample',
                                    'sample_audvis_raw.fif')
raw = mne.io.read_raw_fif(sample_data_raw_file, verbose=False)
raw.crop(tmax=60).load_data()
```

因为样本数据集包含实验事件记录的`stim`通道(`STI 014`)，通过`mne.find_events()`从刺激通道中提取事件：

```python
events = mne.find_events(raw, stim_channel='STI 014')
# 86 events found
# Event IDs: [ 1  2  3  4  5 32]
```

## 选择和拼接事件

`find_event()`的输出为找到的事件的个数，以及唯一的整数事件ID。

可以通过向`mne.pick_events()`函数传入参数`include`和`exclude`来选择感兴趣的事件。例如，样本数据中事件ID32对应被试按下按键，可以通过以下方式选择：

```python
events_no_button = mne.pick_events(events, exclude=32)
```

可以通过`mne.merge_event()`方法将事件ID组合为一个；如下例子将ID1,2,3变为统一的ID`1`：

```python
merged_events = mne.merge_events(events, [1, 2, 3], 1)
print(np.unique(merged_events[:, -1]))
# [1 4 5 32]
```

## 将事件IDs映射到试次描述信息

```python
event_dict = {'auditory/left': 1, 'auditory/right': 2, 'visual/left': 3,
              'visual/right': 4, 'smiley': 5, 'buttonpress': 32}
```

类似上方的事件字典可以在提取试次`epoch`时使用，生成的`Epochs`允许通过部分描述符进行选择。例如，想要选取所有`auditory`试次，因为在`event_dict`中的键中包含`/`字符分隔的多试次描述符：筛选`auditory`试次将会选择所有事件ID为1和2的`epoch`。

## Plotting events

事件字典的另一个用途是在绘制事件时，可以用来检测`STIM`通道上的事件信号是否正确，MNE-Python是否成功发现它们。`mne.viz.plot_events()`函数绘制每个事件及其对应样本点(如果提供频率，事件单位将是秒`s`)。如果提供事件字典，将会生成图例。

```python
fig = mne.viz.plot_events(events, sfreq=raw.info['sfreq'],
                          first_samp=raw.first_samp, event_id=event_dict)
fig.subplots_adjust(right=0.7)  # make room for legend
```

![](https://mne.tools/stable/_images/sphx_glr_20_event_arrays_001.png)

## Plotting events and raw data together

通过向`raw.plot()`传入`events`参数，将事件以及`Raw`对象一同显示：

```python
raw.plot(events=events, start=5, duration=10, color='gray',
         event_color={1: 'r', 2: 'g', 3: 'b', 4: 'm', 5: 'y', 32: 'k'})
```

![](https://mne.tools/stable/_images/sphx_glr_20_event_arrays_002.png)

## 构造等间隔的事件数组

对于分析静息状态活动的实验，在数据记录过程中可能没有任何的实验事件。在这种情况下，可以通过`mne.make_fixed_length_events()`构造等间隔的事件数组。

```python
new_events = mne.make_fixed_length_events(raw, start=5, stop=50, duration=2.)
```

默认情况下，事件会被赋予事件ID为`1`，但可以通过`id`参数更改。也可以指定重叠(`overlap`)持续事件。如果想要2.5s长的`epochs`，重叠0.5秒，可以指定参数`duration=2.5, overlap=0.5`。

