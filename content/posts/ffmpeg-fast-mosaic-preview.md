---
title: "ffmpeg快速生成预览贴图"
date: 2019-09-13 00:00:00 +0800
comments: true
tags: [ ffmpeg ]
---

<!--more-->
搜索下如何用ffmpeg截取视频快照,看到有很多的办法是使用`-vf "select=.."`,这种办法的好处是可以定制更丰富的截屏条件，对桢的控制也更精确，
但是无法忍受的一点是,使用`-vf `,ffmpeg是需要对输入的视频逐帧解码的，想象一个视频只需要在01:00:00处截图一次却要耗费大量cpu把前面所有的帧解一次，
况且笔记本CPU不行解码速度远小于正常播放速度,多文件批处理的情况下等待时间恐怕是要按天来算了

一个更简单的方式是使用`-ss`选项，ffmpeg根据这个传入的时间参数快速定位到指定的帧, 类似于在播放器界面上拖动视频进度条到指定位置的效果  
`-ss`选项对每个文件只能生效一次，能否利用这个选项生成常见的那种视频的多图预览的瓷砖效果呢,当然是可以的:
- 多次对同一个文件使用-ss定位到一个时间N，生成 截屏_N.png
- 把多个文件合并起来，使用tile 或者xstack 等filter把图片合成一张,比如: `ffmpeg -i 截屏_1.png -i 截屏_2.png ... -vf "tile=.."`  
参考[这里](https://stackoverflow.com/questions/24604689/how-to-join-two-images-into-one-with-ffmpeg)

作为一个命令行洁癖患者看到有中间文件生成就感觉不适，能不能在一个命令中完成呢，ffmpeg如此强大，这点需求完全是小意思,
用[filtergraph](https://ffmpeg.org/ffmpeg-filters.html#Filtergraph-description)把各种filter搭配在一起就可以  
做个测试确认ffmpeg是可以接收多个`-ss <TIME> -i <FILE>`这种格式的，相当与一个命令中可以多次读取同一个文件定位到不同的位置作为多个输入，并且在filtergraph中引用这些输入并进一步处理合并

比如下面的命令可以生产12帧的电影截图:
![](/images/ffmpeg1.png)
![](/images/ffmpeg2.png)


每个视频的截图时间是一个列表参数比如[5,60,300,...]秒, 凑出对应的命令是需要一点编程的, 下面就是用来生成以上4x3预览图片的脚本.

{{< highlight bash >}}
#! /bin/bash
FFMPEG="ffmpeg -nostdin -loglevel error -nostats"
INFILE="$1"
OUTFILE="${2:-$1.png}"

[ -z "INFILE" ] && exit 1;

DURATION=$(ffprobe -loglevel error -show_entries format=duration -of default=nw=1:nk=1 "$INFILE")
DURATION=${DURATION%.*}

[[ ! $DURATION =~ ^[0-9]+$ ]] && exit 1

echo "duration is $DURATION s"
NSHOT=12
COL=4
SHOTS=(0)
if [ $DURATION -ge $NSHOT ] ;then
    SEG=$(($DURATION/$NSHOT))
    SHOTS=($(seq 0 $SEG $DURATION))
    SHOTS=(${SHOTS[@]:1})

    if [ $SEG -ge 300 ];then
        SHOTS=(5 30 90 180 ${SHOTS[@]})
    elif [ $SEG -ge 150 ];then
        SHOTS=(5 30 60 100 ${SHOTS[@]})
    elif [ $SEG -ge 60 ];then
        SHOTS=(5 10 20 40 ${SHOTS[@]})
    fi
fi
echo "Screenshots: ${SHOTS[@]}"

SCALE=1000:-1
PADS=([{a..z}] [{A..Z}])
NFRAME=${#SHOTS[@]}

OPT_SEEK=""
OPT_FILTER=""
i=0
for t in ${SHOTS[@]} ;do 
    OPT_SEEK="${OPT_SEEK} -ss $t -i '$INFILE'"
    OPT_FILTER="${OPT_FILTER}[$i:v]trim=start_frame=1:end_frame=2${PADS[$i]};"
    i=$((++i))
done
OPT_FILTER="${OPT_FILTER}${PADS[@]:0:$i}concat=n=$i,tile=${COL}x$(($NFRAME/$COL+(($NFRAME%$COL)!=0))),scale=$SCALE"

eval "$FFMPEG $OPT_SEEK -filter_complex '$OPT_FILTER' -y '$OUTFILE'"
{{< /highlight >}}

一点说明:
- 要定制截图的多个时间点需要先得到视频总时长，这里用`ffprobe`来获取,参考[这里](https://superuser.com/questions/650291/how-to-get-video-duration-in-seconds)
- 原则上把视频按照截图数N拆时间间隔，但是我觉得影片前面5-10分钟更重要所以额外定制前面4帧
- 这种截屏的好处是运行很快，生成图片过程基本是按秒来计, 一点不足是据说`-ss`的时间定位并不完全精确但也够用了
