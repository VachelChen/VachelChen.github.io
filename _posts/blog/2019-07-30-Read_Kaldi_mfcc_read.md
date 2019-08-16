---
layout: post
title: Kaldi 文本分析 —— MFCC Features
categories: Kaldi
description: 分析与提取特征相关的脚本文件，与理论知识做比较。
keywords: Kaldi, mfcc, features
---

run.sh的第二块内容就是创建提取mfcc特征：

```shell

#produce MFCC features

rm -rf data/mfcc && mkdir -p data/mfcc &&  cp -R data/{train,dev,test,test_phone} data/mfcc || exit 1;

#这里同样和数据准备类似，要先删除原有的mfcc特征，并且递归拷贝数据准备创建的几个文件到mfcc下。

for x in train dev test; do

   #make  mfcc

   steps/make_mfcc.sh --nj $n --cmd "$train_cmd" data/mfcc/$x exp/make_mfcc/$x mfcc/$x || exit 1;

   # 分别对训练集、验证集、测试集创建MFCC模型，以下将对make_mfcc.sh进行分析。

   #compute cmvn

   steps/compute_cmvn_stats.sh data/mfcc/$x exp/mfcc_cmvn/$x mfcc/$x || exit 1;
	
   # 理论中设计的倒谱一概念，在这里需要对mfcc三个集合中的倒谱均值和方差归一化 。
done

```

---

## 一、make_mfcc.sh

我原本以为这个文件会与上次所分析的理论对应
但其实分析脚本下来还是一些和文件处理相关的

> 这里说明一下shell的参数具体是这样的：
> $# 表示提供到shell脚本或者函数的参数总数；
> $1 表示第一个参数。
> -ne 表示 不等于
> 另外：整数比较-eq 等于,如:if ["$a" -eq "$b" ]
> -ne 不等于,如:if ["$a" -ne "$b" ]
> -gt 大于,如:if ["$a" -gt "$b" ]
> -ge大于等于,如:if ["$a" -ge "$b" ]
> -lt 小于,如:if ["$a" -lt "$b" ]
> -le 小于等于,如:if ["$a" -le "$b" ]
> < 小于(需要双括号),如:(("$a" < "$b"))
> <= 小于等于(需要双括号),如:(("$a" <= "$b"))
> ‘>‘ 大于(需要双括号),如:(("$a" "$b"))
> ‘>=’ 大于等于(需要双括号),如:(("$a" >= "$b"))
> 另外：$?是shell变量,表示"最后一次执行命令"的退出状态.0为成功,非0为失败.

文件如下：

```shell

#!/bin/bash

# Copyright 2012-2016  Johns Hopkins University (Author: Daniel Povey)
# Apache 2.0
# To be run from .. (one directory up from here)
# see ../run.sh for example

# Begin configuration section.
nj=4
# 共同工作进程数
cmd=run.pl
# 调用run.pl
mfcc_config=conf/mfcc.conf
#十二个系数以外的能量要不要
#已注释抽样率 8000 足够

compress=true
# 启用了压缩
write_utt2num_frames=false  # if true writes utt2num_frames

# End configuration section.
# 打印这个脚本的名称以及所有的参数
echo "$0 $@"  # Print the command line for logging
# 这里是打印命令行到日志，$0为执行的命令 $@表示所有参数脚本的内容

# 加载path.sh和parse_options.sh
if [ -f path.sh ]; then . ./path.sh; fi #设置环境变量
# 如果存在path.sh那么就执行它

. parse_options.sh || exit 1;
# parse_options.sh解析命令行选项

# 如果参数少于1或者大于3就提示使用脚本错误

if [ $# -lt 1 ] || [ $# -gt 3 ]; then
  #如果参数小于一个或大于三个 标准错误输出 注释
   echo "Usage: $0 [options] <data-dir> [<log-dir> [<mfcc-dir>] ]"; #第二个是日志
   echo "e.g.: $0 data/train exp/make_mfcc/train mfcc"
   echo "Note: <log-dir> defaults to <data-dir>/log, and <mfccdir> defaults to <data-dir>/data"
   echo "Options: "
   echo "  --mfcc-config <config-file>                      # config passed to compute-mfcc-feats "
   echo "  --nj <nj>                                        # number of parallel jobs"
   echo "  --cmd (utils/run.pl|utils/queue.pl <queue opts>) # how to run jobs."
   echo "  --write-utt2num-frames <true|false>     # If true, write utt2num_frames file."
   exit 1;
fi
# 这块就是如果不符合这些参数列表，则需要更改符合这些参数。

data=$1		    # 将第一个参数赋给data，其实就是上面这个说明中说的<data-dir>c/train
# 判断如果参数总数大于等于2，则将第二个参数赋给logdir，否则将data下的log路径赋给logdir
if [ $# -ge 2 ]; then	# 如果参数大于等于2
  logdir=$2		# logdir=exp/make_mfcc/train
else
  logdir=$data/log
fi
if [ $# -ge 3 ]; then	# 判断如果参数总数大于等于3，则将第三个参数赋给mfccdir，否则将data下的data路径赋给mfccdir
  mfccdir=$3		#mfccdir=mfcc/train
else
  mfccdir=$data/data
fi

# make $mfccdir an absolute pathname.
mfccdir=`perl -e '($dir,$pwd)= @ARGV; if($dir!~m:^/:) { $dir = "$pwd/$dir"; } print $dir; ' $mfccdir ${PWD}`
# -e 执行后面的语句
#  
#  ！否定 ~匹配 m ^开头 /代表根目录
#  若不是绝对路径改成
#utterence

# use "name" as part of name of the archive.
name=`basename $data`	# data/mfcc/train输出train
# 创建mfcc特征文件夹和log文件夹
mkdir -p $mfccdir || exit 1;
mkdir -p $logdir || exit 1;
# 如果之前有执行过生成了特征信息文件则备份
if [ -f $data/feats.scp ]; then
  mkdir -p $data/.backup
  echo "$0: moving $data/feats.scp to $data/.backup"
  mv $data/feats.scp $data/.backup
fi

scp=$data/wav.scp	# 得到音频路径列表

required="$scp $mfcc_config"

for f in $required; do		# 检测wav.scp和mfcc_config.sh文件是否存在
  if [ ! -f $f ]; then
    echo "make_mfcc.sh: no such file $f"
    exit 1;
  fi
done

# 使用validate_data_dir.sh 检测$data里的内容是否正确

utils/validate_data_dir.sh --no-text --no-feats $data || exit 1;
if [ -f $data/spk2warp ]; then
  echo "$0 [info]: using VTLN warp factors from $data/spk2warp"
  vtln_opts="--vtln-map=ark:$data/spk2warp --utt2spk=ark:$data/utt2spk"
elif [ -f $data/utt2warp ]; then
  echo "$0 [info]: using VTLN warp factors from $data/utt2warp"
  vtln_opts="--vtln-map=ark:$data/utt2warp"
fi

for n in $(seq $nj); do	# 几个线程就分几个文件 .ark中存放音频mfcc特征
  # the next command does nothing unless $mfccdir/storage/ exists, see
  # utils/create_data_link.pl for more info.
  utils/create_data_link.pl $mfccdir/raw_mfcc_$name.$n.ark
done


if $write_utt2num_frames; then
  write_num_frames_opt="--write-num-frames=ark,t:$logdir/utt2num_frames.JOB"
else
  write_num_frames_opt=
fi

if [ -f $data/segments ]; then	# 如果存在segments文件则使用已有文件
  echo "$0 [info]: segments file exists: using that."

  split_segments=""
  for n in $(seq $nj); do
    split_segments="$split_segments $logdir/segments.$n"
  done

  utils/split_scp.pl $data/segments $split_segments || exit 1;
  rm $logdir/.error 2>/dev/null

  $cmd JOB=1:$nj $logdir/make_mfcc_${name}.JOB.log \
    extract-segments scp,p:$scp $logdir/segments.JOB ark:- \| \
    compute-mfcc-feats $vtln_opts --verbose=2 --config=$mfcc_config ark:- ark:- \| \
    copy-feats --compress=$compress $write_num_frames_opt ark:- \
      ark,scp:$mfccdir/raw_mfcc_$name.JOB.ark,$mfccdir/raw_mfcc_$name.JOB.scp \
     || exit 1;

else	# 我使用的时候执行此分支
  echo "$0: [info]: no segments file exists: assuming wav.scp indexed by utterance."
  split_scps=""
  for n in $(seq $nj); do
    split_scps="$split_scps $logdir/wav_${name}.$n.scp"
    # 后面是 exp/make_mfcc/train/wav_train.1.scp
  done

  utils/split_scp.pl $scp $split_scps || exit 1;	# 使用脚本处理 scp=$data/wav.scp


  # add ,p to the input rspecifier so that we can just skip over
  # utterances that have bad wave data.
	# 这里用run.pl提取特征开始
  $cmd JOB=1:$nj $logdir/make_mfcc_${name}.JOB.log \
    compute-mfcc-feats  $vtln_opts --verbose=2 --config=$mfcc_config \
     scp,p:$logdir/wav_${name}.JOB.scp ark:- \| \
      copy-feats $write_num_frames_opt --compress=$compress ark:- \
      ark,scp:$mfccdir/raw_mfcc_$name.JOB.ark,$mfccdir/raw_mfcc_$name.JOB.scp \
      || exit 1;
fi
#最后生成的应该就是mfcc/train 中的raw_mfcc_train.1.ark raw_mfcc_train.1.scp

if [ -f $logdir/.error.$name ]; then	# 如果出现了错误则打印出log中最后的错误信息
  echo "Error producing mfcc features for $name:"
  tail $logdir/make_mfcc_${name}.1.log
  exit 1;
fi

# concatenate the .scp files together.
for n in $(seq $nj); do
  cat $mfccdir/raw_mfcc_$name.$n.scp || exit 1;
done > $data/feats.scp || exit 1	# 将所有的scp文件拼接起来输出到data/mfcc/train/feats.scp

if $write_utt2num_frames; then
  for n in $(seq $nj); do
    cat $logdir/utt2num_frames.$n || exit 1;
  done > $data/utt2num_frames || exit 1
  rm $logdir/utt2num_frames.*
fi
# 删除过程文件
rm $logdir/wav_${name}.*.scp  $logdir/segments.* 2>/dev/null

nf=`cat $data/feats.scp | wc -l`	# 输出文件的行数
nu=`cat $data/utt2spk | wc -l`
if [ $nf -ne $nu ]; then	# 检测特征的数目与音频文件的数目是否相同
  echo "It seems not all of the feature files were successfully processed ($nf != $nu);"
  echo "consider using utils/fix_data_dir.sh $data"
fi

if [ $nf -lt $[$nu - ($nu/20)] ]; then
  echo "Less than 95% the features were successfully generated.  Probably a serious error."
  exit 1;
fi

echo "Succeeded creating MFCC features for $name"


```

![filter](/images/blog/filter.png)

预加重是使用滤波器完成的，即先过滤出高频的声段，再进行加重操作，滤波器如下所示：

![filter2](/images/blog/filter2.png)

式中的值介于0.9-1.0之间，我们通常取0.97。

---

## 二、加窗（Windowing）

语音特征其实有很大的变化，我们在不同的情绪以及不同的环境中所收集的所谓频谱特征（spectral features）是不同的，所以我们认为语音其实是一个非平稳信号，这意味着他的统计特性在时间上不是恒定的。但我们一般认为在短时间内提取的频谱特征是平稳的。

![windows](/images/blog/windows.png)

我们通过在某区域内使用窗口来提取波形，同时使用窗口的宽度 wide（以毫秒为单位）、连续窗口之间的偏移量 offset 以及窗口的形状 shape 来描述窗口过程，窗口提取到的语音称为帧，其中帧的毫秒值称为帧的大小 frame size （一般为25ms），并将连续窗口的左边缘间隔的毫秒值称之为帧的移动 （一般为10ms）frame shift。方程为：

```
y[n] = w[n]s[n]
```

其中n表示时间，s[n]表示语音原本的状态，w[n]表示窗口形状的处理，y[n]表示最终状态。

窗口形状的概念听起来比较抽象，以下例子将做一个说明。
上图中窗口提取出来的信号与原始信号一样，所以这些窗口都是矩形窗口，也是最简单的窗口。然而，矩形窗口可能会引起问题，因为它是垂直的，会在其边界处突然切断信号。当我们做傅里叶分析时，这些不连续点会产生问题。因此，在MFCC提取中使用的一个更常见的窗口是汉明窗口（Hamming Window），它将信号的值在窗口边界处缩小到汉明零，以避免不连续。方程如下(假设窗口为L帧长):

![windows2](/images/blog/windows2.png)

以下是一般情况的汉明窗，不同的a值会产生不同的汉明窗，一般情况下a取0.46。

![windows3](/images/blog/windows3.png)

汉明窗得到的信号：

![windows4](/images/blog/windows4.png)

### 离散傅里叶变换（Discrete Fourier Transform）

乍一听到傅里叶变换感觉熟悉又陌生，这里先来简单复习一下。
傅里叶变换，表示能将 满足一定条件的 某个函数 表示成三角函数（正弦和/或余弦函数） 或者它们的 积分的线性组合。

![FT](/images/blog/FT.png)

而离散傅里叶变换（DFT）是傅里叶变换在时域和频域上都呈离散的形式，在这里我们把一个个窗体作为输入，输出是一个个复数，表示信号中的频率分量的大小和相位，
计算过程比较复杂，但我们通常使用欧拉公式帮助我们分析：

![Euler](/images/blog/Euler.png)

DFT定义如下：

![dft](/images/blog/dft.png)

计算DFT的一种常用算法是快速傅里叶变换(FFT)，用于实现DFT十分有效，但只适用于FFF的N值是2的幂。

---

### 倒谱（The Cepstrum）

这里有一个有趣的文字游戏，频谱的前四个字母倒过来就是倒谱：

频谱 spectrum

倒谱 cepstrum


因为梅尔谱（Mel spectrum）来做特征提取会存在一定问题，为了解决这一点，我们下一步将进行倒谱（cepstrum）计算。

倒谱可以理解为频谱取log的频谱(spectrum of the log of the spectrum)，这句话我读的时候感觉不是人话，图形画出来大致是这样的：

![spectrum](/images/blog/spectrum.png)

其中（a）是频谱的样子，(b)是频谱取log的样子，（c）是倒谱的样子。

倒谱系数具有一个非常有用的性质，不同系数的变量是趋于不相关的，而在频谱中不是这样的，这意味着高斯混合模型GMM不必表示所有MFCC特征之间的协方差，减少了参数的数量。对于一个有窗口的语音帧x[n]，经过如下的步骤成倒谱：

![spectrum2](/images/blog/spectrum2.png)

对于MFCC的提取，我们通常只取前12个值。

---

### 能量（Deltas and Energy）

每帧得到12个倒谱系数后，能量是第13个特征。对于时间样本t1到时间样本t2窗口内的信号x，能量为:

![energy](/images/blog/energy.png)

因为帧（frame）与帧之间不是恒定的，我们还需要添加与倒谱特性随时间变化的相关特性给13个特征（12个倒谱系数+能量），这个特性称 delta 或速度、double delta 或加速度。

> 我理解13个delta特征为：帧与帧之间的变化。

> 我们理解13个double delta特征为：帧之间的变化的变化（类似于加速度，描述速度的变化）。

算delta最简单的方法：

![delta](/images/blog/delta.png)

---

### MFCC

添加能量后，39个MFCC特征就齐了：

```
 12 个倒谱系数
 12 个倒谱系数变化速率
 12 个倒谱系数变化速率的速率
 1  个能量系数
 1  个变化速率的能量系数
 1  个变化速率的速率的能量系数
 ————————————————————————————————
 39 MFCC 特征
```

同样，关于MFCC特征最有用的事实之一是倒谱系数往往是不相关的，这将使我们的声学模型更简单。