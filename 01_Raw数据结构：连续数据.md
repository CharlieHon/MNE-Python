#mne #eeg #python 

## 导入连续数据

默认情况下，`mne.io.read_raw_*`一些列函数不会将数据加载到内存上，而是只能从硬盘中读取。一些运算如滤波需要将数据复制到`RAM`中，可以通过给函数传递`preload=True`参数来实现。也可以通过使用`load_data()`方法随时将数据加载到`RAM`中。

通过`print(raw)`可以看到数据的一些基本信息，如通道数量、数据采样点数、时间长度和占用内存大小
。更多信息可以通过`Raw`类的各种属性和方法获得，如通过`ch_names`获取通道名称的列表，通过`times`可以获得以秒为单位的样本点，通过`n_times`获得采样点的总数。

## 查询`Raw`对象

### `Raw.info` 属性

`raw.info`类似于`Python dictionary`，存储数据的很多信息。通过`.keys()`方法可以显示可以获得的字段名称。而打印`print(raw.info)`可以输出格式化的每个字段名的数据信息。

```python
n_time_samps = raw.n_times   # 采样点
time_secs = raw.times        # 时间点(s)
ch_names = raw.ch_names      # 通道名称
n_chan = len(ch_names)  # 通道个数，注意：没有专门获得通道个数的属性
print('the (cropped) sample data object has {} time samples and {} channels.'
      ''.format(n_time_samps, n_chan))
print('The last time sample is at {} seconds.'.format(time_secs[-1])) # 时长(s)
print('The first few channel names are {}.'.format(', '.join(ch_names[:3])))

# some examples of raw.info:
print('bad channels:', raw.info['bads'])  # 坏导
print(raw.info['sfreq'], 'Hz')            # 采样率
print(raw.info['description'], '\n')      # 各种描述信息

print(raw.info)
```

### 时间与采样点

`time_as_index()`是`Raw`对象中频繁使用的一个方法，可以**将以`s`为单位的时间点转换为最接近该时间的样本的整数索引值**。

因为对应的时间可能没有采样点，所以时间`time=1`和`time=2`之间的采样点个数可能与`time=2`到`time=3`之间的数量不同。

```python
print(raw.time_as_index(20)) # [12012]
print(raw.time_as_index([20, 30, 40]), '\n') # [12012 18018 24024]
print(np.diff(raw.time_as_index([1, 2, 3]))) # [601 600] 个数不同
```

## 更改 `Raw` 对象

`Raw`对象有很多方法会就地更改`Raw`对象本身，并返回被修改后的实例。为防止修改后又要对原数据进行处理，可以通过`copy()`方法在修改数据之前进行复制，这样原始数据仍然可以通过`raw`变量获得。

### 选择、丢弃以及重新排序通道

可以通过多种方法选择`Raw`对象的通道，如使用`pick_types()`方法限定到`Raw`对象的EEG和EOG通道。

```python
eeg_and_eog = raw.copy().pick_types(meg=False, eeg=True, eog=True)
```

`pick_channels()`方法可以通过通道名称指定特定通道的数据，对应的`drop_channels()`方法通过名称去除指定通道数据。

```python
raw_temp = raw.copy()
print('Number of channels in raw_temp:')
print(len(raw_temp.ch_names), end=' → drop two → ')
raw_temp.drop_channels(['EEG 037', 'EEG 059'])
print(len(raw_temp.ch_names), end=' → pick three → ')
raw_temp.pick_channels(['MEG 1811', 'EEG 017', 'EOG 061'])
print(len(raw_temp.ch_names))

# Number of channels in raw_temp:
# 376 → drop two → 374 → pick three → 3
```

如果要通道以特定顺序排列，可以使用`reorder_channels()`方法，它的功能和`pick_channels()`类似，可以选择指定通道，并且还能重新将通道排序。如下，提取EOG和前额EEG通道并将EEG通道逆序排列：

```python
channel_names = ['EOG 061', 'EEG 003', 'EEG 002', 'EEG 001']
eog_and_frontal_eeg = raw.copy().reorder_channels(channel_names)
print(eog_and_frontal_eeg.ch_names)
# ['EOG 061', 'EEG 003', 'EEG 002', 'EEG 001']
```

### 更改通道名称和类型

通过向`rename_channels()`方法，传入一个Python字典，将想要更改的原名字映射到新名字即可。

```python
raw.rename_channels({'EOG 061': 'blink detector'})
```

以下例子，将使用Python字典推导式将数据后三个通道名称中的空格替换成下划线`_`。

```python
print(raw.ch_names[-3:])
channel_renaming_dict = {name: name.replace(' ', '_') for name in raw.ch_names}
raw.rename_channels(channel_renaming_dict)
print(raw.ch_names[-3:])

# ['EEG 059', 'EEG 060', 'blink detector']
# ['EEG_059', 'EEG_060', 'blink_detector']
```

可以通过`set_channel_types()`方法更改通道的类型。该方法接受将通道名称映射到类型的Python字典，允许的类型有`ecg, eeg, emg, eog, ecog, stim`等。经常会将前额EEG通道更改为为EOG通道。

```python
raw.set_channel_types({'EEG_001': 'eog'})
print(raw.copy().pick_types(meg=False, eog=True).ch_names)
# ['EEG_001', 'blink_detector']
```

### 选择时间范围

如果想限制`Raw`对象的时间范围，可以使用`crop()`方法，就地修改`Raw`对象。函数参数有`tmin`和`tmax`，都是以秒`s`为单位。以下先调用`copy()`防止修改原数据：

```python
raw_selection = raw.copy().crop(tmin=10, tmax=12.5)
print(raw_selection)

# <Raw | sample_audvis_raw.fif, 376 x 1503 (2.5 s), ~3.3 MB, data not loaded>
```

`crop()` 剪切操作会更改`first_samp`和`times`属性，剪切后的对象第一个样本点会从零开始，即`time=0`。因此，如果想再次剪切数据的话，如从11到12.5秒的数据。传入的参数应该是`tmin=1`(而非`time=11`)，不指定`tmax`就会从`tmin`开始剪切到数据末尾。

```python
print(raw_selection.times.min(), raw_selection.times.max())  # 0.0 2.500770084699155
raw_selection.crop(tmin=1)
print(raw_selection.times.min(), raw_selection.times.max())  # 0.0 1.5001290587975622
```

因为采样的原因，样本点并不是总和需要的`tmin`和`tmax`值确切相匹配。如果想要选择非连续时间片段的数据或者拼接2个或更多独立`Raw`对象，可以使用`append()`方法：

```python
raw_selection1 = raw.copy().crop(tmin=30, tmax=30.1)     # 0.1 seconds
raw_selection2 = raw.copy().crop(tmin=40, tmax=41.1)     # 1.1 seconds
raw_selection3 = raw.copy().crop(tmin=50, tmax=51.3)     # 1.3 seconds
raw_selection1.append([raw_selection2, raw_selection3])  # 2.5 seconds total
print(raw_selection1.times.min(), raw_selection1.times.max())  # 0.0 2.5041000049184614
```

## 从`Raw`对象中提取数据

本节将会展示如何从`Raw`对象中提取数据(`Numpy array`格式)，为方便后续在其它库中使用分析或绘图函数。可以通过方括号`[]`进行索引，选择数据的一部分。`Raw`对象索引与`Numpy array`数组有两点不同：
1. MNE-Python会以元组的形式返回需要的样本值以及对应的时间点(以`s`为单位)，即(data, times)。
2. 无论选取一个时间点还是一个通道，数据数组总是2维的。

### 通过索引(index)提取数据

为了演示上述两点，以下从第一个通道选取2s的数据：

```python
sampling_freq = raw.info['sfreq']
start_stop_seconds = np.array([11, 13])
start_sample, stop_sample = (start_stop_seconds * sampling_freq).astype(int)
channel_index = 0
raw_selection = raw[channel_index, start_sample:stop_sample]
print(raw_selection)
# (array(...), array(...))
```

可以看到返回值包含两个数组，这种将数据和时间点结合的方式方便绘制选定数据。如下将数据转置，是为了使每个通道的数据呈一列而非一行，匹配matplotlib以一维的时间点和二维数据作为参数时的参数格式要求。

```python
times = raw_selection[1]
data = raw_selection[0].T
plt.plot(times, data)
```

![](https://mne.tools/stable/_images/sphx_glr_10_raw_overview_001.png)

### 通过名称提取通道

`Raw`对象也可以通过通道名称索引。可以传递一个字符串选择一个通道，也可以通过一个字符串列表选择多个通道。与数值索引一样，返回值是一个元组`(data_array, times_array)`。如下选择两个通道，这里我们将在一个频道上添加一个垂直偏移量，这样两个通道数据就不会重叠在一起了。

```python
channel_names = ['MEG_0712', 'MEG_1022']
two_meg_chans = raw[channel_names, start_sample:stop_sample]
y_offset = np.array([5e-11, 0])  # just enough to separate the channel traces
x = two_meg_chans[1]
y = two_meg_chans[0].T + y_offset
lines = plt.plot(x, y)
plt.legend(lines, channel_names)
```

![](https://mne.tools/stable/_images/sphx_glr_10_raw_overview_002.png)

### 通过类型(type)选择通道

有多种方法可以从`Raw`对象中选择同一类型的所有通道的数据。最安全的方法是使用`mne.pick_types()`获得想要通道的**整数索引**，然后用这些索引通过方括号`[]`获取数据。`pick_types()`函数使用`Raw`对象的`Info`属性来确定通道的类型，通过布尔值或者字符串参数来标注想要保留的通道类型。默认情况下，`meg`参数为`True`，所有其它类型默认值为`False`，所以为了只得到EEG通道，需要传递参数`eeg=True` **和**`meg=False`两个参数：

```python
eeg_channel_indices = mne.pick_types(raw.info, meg=False, eeg=True)
eeg_data, times = raw[eeg_channel_indices]
print(eeg_data.shape)  # (58, 36038)
```

### `Raw.get_data()`方法

如果只想要数据而不包括对应的时间点，`Raw`对象有一个`get_data()`方法。当不传入任何参数时，它将会提取所有通道的所有数据，返回一个`(n_channels, n_timepoints)`大小的`Numpy array`：

```python
eeg_channel_indices = mne.pick_types(raw.info, meg=False, eeg=True)
eeg_data, times = raw[eeg_channel_indices]
print(eeg_data.shape)  # (376, 36038)
```

如果想要返回时间点，可以使用可选参数`return_times`。

```python
data, times = raw.get_data(return_times=True)
print(data.shape)  # (376, 36038)
print(times.shape) # (36038, )
```

`get_data()`方法也可以通过`picks`，`start`和`stop`参数，提取特定的通道和样本范围。`picks`参数接受整数通道索引、通道名称或者通道类型作为参数，提取按照参数顺序排列的数据。

```python
first_channel_data = raw.get_data(picks=0)  # 整数索引
eeg_and_eog_data = raw.get_data(picks=['eeg', 'eog'])  # 通道类型
two_meg_chans_data = raw.get_data(picks=['MEG_0712', 'MEG_1022'],
                                  start=1000, stop=2000)  # 通道名称

print(first_channel_data.shape)  # (1, 3638)
print(eeg_and_eog_data.shape)    # (61, 36038)
print(two_meg_chans_data.shape)  # (2, 1000)
```

### 从`Raw`对象中提取数据总结

如下表格总结了从`Raw`对象中提取数据的各种方法。

| Python code                                                      | Result                                               |
| ---------------------------------------------------------------- | ---------------------------------------------------- |
| `raw.get_data()`                                                 | `Numpy array` (n_chans×n_samps)                      |
| `raw[:]`                                                         | `tuple` of (data(n_chans×n_samps), times(1×n_samps)) |
| `raw.get_data(return_times=True)`                                |                                                      |
| `raw[0, 1000:2000]`                                              | `tuple` of (data(1×1000), times(1×1000))             |
| `raw['MEG 0113', 1000:2000]`                                     |                                                      |
| `raw.get_data(pick=0, start=1000, stop=2000, return_times=True]` |                                                      |
| `raw.get_data(picks='MEG 0113', start=1000, stop=2000, return_times=True)` |                                          |
| `raw[7:9, 1000:2000]`                                            |   `tuple` of (data(2×1000), times(1×1000))                                                   |
| `raw[[2, 5], [1000:2000]`                                        |                                                      |
| `raw[['EEG 030', 'EOG 061'], 1000:2000]`                         |                                                      |

## 导出和保存`Raw`对象

`Raw`对象有个内置的`save()`方法，可以用来保存处理后的数据，以便后续可以重新加载，同时各种属性完好无损。

还有一些方法可以仅从`Raw`对象导出传感器数据(即通道数据)。一个是使用索引或`get_data()`方法来提取数据，然后使用`numpy.save()`将数据保存为`.npy`格式：

```python
data = raw.get_data()
np.save(file='my_data.npy', arr=data)
```

可以将数据导出为`Pandas DataFrame`对象，然后用`Pandas`保存方法。`Raw`对象的`tp_data_frame()`方法类似于`get_data()`方法，有`picks`、`start`和`stop`等参数。但是注意默认会将时间单位转变为毫秒(ms)。

```python
sampling_freq = raw.info['sfreq']
start_end_secs = np.array([10, 13])
start_sample, stop_sample = (start_end_secs * sampling_freq).astype(int)
df = raw.to_data_frame(picks=['eeg'], start=start_sample, stop=stop_sample)
# then save using df.to_csv(...), df.to_hdf(...), etc
print(df.head())
```

```markdown
		time  ...       EEG_060
0   9.999750  ...  6.952283e+08
1  10.001415  ...  7.069226e+08
2  10.003080  ...  7.080921e+08
3  10.004745  ...  7.010755e+08
4  10.006410  ...  7.069226e+08

[5 rows x 60 columns]
```
