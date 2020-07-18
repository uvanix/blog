JENKINS推送脚本
```
echo "清除当前分支代码所有修改"
ssh dev34 'cd /data/workspace/project-name; git checkout .; git clean -df'
echo "切换分支，拉取最新代码"
ssh dev34 'cd /data/workspace/project-name; git checkout dev; git pull'
 
echo "修改配置"
ssh dev34 "sed -i 's/mongodb.test.com:27020/mongodb.test.com:27015/g' /home/workspace/project-name/application.yml"
 
echo "开始构建打包JAVA应用..."
ssh dev34 'cd /data/workspace/project-name; export JAVA_HOME=/opt/jdk1.8.0_131; export PATH=$PATH:${JAVA_HOME}/bin; export MAVEN_HOME=/opt/apache-maven-3.6.0; export PATH=$PATH:${MAVEN_HOME}/bin; mvn clean package -U -X -Dmaven.test.skip=true'
 
echo "杀掉应用..."
ssh dev34 "netstat -ntlp | grep 8080 | grep tcp | awk '{print \$7}' | awk -F/ '{print \$1}' | xargs -r kill; sleep 5"
#echo "清除日志"
 
echo "启动JAVA应用..."
ssh dev34 'cd /data/workspace/project-name; export JAVA_HOME=/opt/jdk1.8.0_131; export PATH=$PATH:${JAVA_HOME}/bin; nohup java -Duser.timezone=Asia/Shanghai -Djava.awt.headless=true -Djava.net.preferIPv4Stack=true -jar project-name.jar --spring.config.location=application.yml --spring.profiles.active=dev > /dev/null 2>&1 &'
 
echo "稍等片刻，程序正在后台启动。。。"
```

FFMPEG加密切片
```
#!/bin/sh
 
FILE=$1
ID=$2
 
openssl rand  16 > "$ID.key"
IV="$(openssl rand -hex 16)"
echo -e "$ID.key\n$ID.key\n$IV" > "$ID.keyinfo"
 
ffmpeg -y \
-i "$FILE" \
-c copy -bsf:v h264_mp4toannexb \
-hls_time 10 \
-start_number 0 \
hls_allow_cache 1 \
-hls_playlist_type vod \
-hls_key_info_file "$ID.keyinfo" \
-hls_segment_filename "$ID_%04d.ts" \
"$ID.m3u8"
```

M3U8下载合并
```
#!/bin/sh
 
if [ $# != 1 ]; then
echo "USAGE: $0 m3u8_url"
exit 1;
fi
 
m3u8_url=$1
m3u8_name=${m3u8_url##*/}
m3u8_dirname=${m3u8_name%.*}
echo "$m3u8_url"
echo "$m3u8_name"
echo "$m3u8_dirname"
mkdir -p $m3u8_dirname
cd $m3u8_dirname
 
echo "Download m3u8"
#echo "wget -O $m3u8_name $m3u8_url"
#wget -O $m3u8_name $m3u8_url
# 使用curl下载支持socks代理
echo "curl --socks5 127.0.0.1:7070 -k -o $m3u8_name $m3u8_url"
curl --socks5 127.0.0.1:7070 --retry 3 -s -k -o $m3u8_name $m3u8_url
 
prefix_url=${m3u8_url%/*}
echo "$prefix_url"
 
echo "Download segments"
cat $m3u8_name | while read line
do
  if echo "$line" | grep ".ts$" > /dev/null; then
    ts_path=$(echo ${line} | grep "/")
    if [[ "$ts_path" != "" ]]; then
      mkdir -p ${line%/*}
    fi
    ts_name=${line##/*}
    #echo "wget -O $ts_name $prefix_url/$line"
    #wget -O $ts_name $prefix_url/$line
    # 使用curl下载支持socks代理
    echo "curl --socks5 127.0.0.1:7070 --retry 3 -s -k -o $ts_name $prefix_url/$line"
    curl --socks5 127.0.0.1:7070 --retry 3 -s -k -o $ts_name $prefix_url/$line
  fi
done
 
# ffmpeg合并m3u8为mp4
cd $m3u8_name && ffmpeg -i "$m3u8_name.m3u8" -c:v copy -c:a copy -bsf:a aac_adtstoasc $m3u8_name.mp4
 
echo "Done"
```

m3u8合并下载（有重试）
```
#!/bin/bash

M3U8_URL=$1
if [[ -z ${M3U8_URL} ]]; then
  echo "USAGE: $0 m3u8_url work_path"
  exit 1
fi
WORK_PATH=$2
if [[ -z ${WORK_PATH} ]]; then
  WORK_PATH=$(pwd)
fi
WORK_PATH="$WORK_PATH/tmpdir"
mkdir -p ${WORK_PATH}

# 下载m3u8
M3U8_NAME=${M3U8_URL##*/}
M3U8_DIRNAME=${WORK_PATH}/${M3U8_NAME%.*}
mkdir -p ${M3U8_DIRNAME}
cd ${M3U8_DIRNAME}
curl --retry 5 -s -k -o ${M3U8_NAME} ${M3U8_URL}
PREFIX_URL=${M3U8_URL%/*}
INDEX=1
cat ${M3U8_NAME} | while read line
do
  if echo "$line" | grep ".ts$" > /dev/null; then
    ((INDEX++))
    ts_path=$(echo ${line} | grep "/")
    if [[ "$ts_path" != "" ]]; then
      mkdir -p ${line%/*}
    fi
    ts_name=${line##/*}
    idx=$((INDEX%2))
    if [[ ${idx} == 0 ]]; then
      curl --retry 3 -s -k -o ${ts_name} ${PREFIX_URL}/${line} &
    else
      curl --retry 3 -s -k -o ${ts_name} ${PREFIX_URL}/${line}
    fi
  fi
done
# 休眠10s 保证最后一个ts下载完成
sleep 10
# 检查ts文件数量
M3U8_FILE_TS_COUNT=$(($(cat ${M3U8_NAME} | grep ".ts$" | wc -l)+1))
M3U8_TS_FILE_COUNT=$(($(ls -lR ${M3U8_DIRNAME} | grep "^-" | wc -l)+0))
if [[ ${M3U8_FILE_TS_COUNT} != ${M3U8_TS_FILE_COUNT} ]]; then
  echo "curl down m3u8 ts($M3U8_FILE_TS_COUNT) size($M3U8_TS_FILE_COUNT) error. rm -rf ${M3U8_DIRNAME}"
  rm -rf ${M3U8_DIRNAME}
  exit 1
else
  echo "curl down m3u8 ts success. m3u8+ts size $M3U8_TS_FILE_COUNT."
fi

WORK_PATH="$M3U8_DIRNAME"
# 生成mp4文件名称
PATH_ENCRYPT_SALT="WvbRv72OBshW50plNB"
MP4_FILE_NAME="$(echo -n "$M3U8_URL$PATH_ENCRYPT_SALT"|md5sum|cut -d ' ' -f1).mp4"
MP4_URL="$WORK_PATH/$MP4_FILE_NAME"

# ffmpeg合并ts为mp4命令
FFMPEG_PATH=$(which ffmpeg)
#${FFMPEG_PATH} -v error -y -i ${M3U8_URL} ${MP4_URL}
${FFMPEG_PATH} -v error -y -i ${M3U8_NAME} ${MP4_URL}

# 校验时长5秒内误差正常
FFPROBE_PATH=$(which ffprobe)
M3U8_DURATION=$(${FFPROBE_PATH} -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 -i ${M3U8_NAME})
MP4_DURATION=$(${FFPROBE_PATH} -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 -i ${MP4_URL})
DIFF_DURATION=$(echo "$M3U8_DURATION $MP4_DURATION" | awk '{printf("%.0f\n",$1-$2) }')
if [[ ${DIFF_DURATION} -gt 5 ]]; then
  echo "ffmpeg merge m3u8($M3U8_DURATION) to mp4($MP4_DURATION) error. rm -rf ${WORK_PATH}"
  rm -rf ${WORK_PATH}
  exit 1
else
  echo "ffmpeg merge m3u8($M3U8_DURATION) to mp4($MP4_DURATION) success."
fi
```

S3-Ceph上传脚本
```
#!/bin/bash

# 根据文件扩展名获取Content-Type(Mime-Type)
contentType() {
  if [ "$FILE_EXTENSION" = "bmp" ]; then
    echo "image/bmp"
  elif [ "$FILE_EXTENSION" = "jpg" ]; then
    echo "image/jpeg"
  elif [ "$FILE_EXTENSION" = "jpeg" ]; then
    echo "image/jpeg"
  elif [ "$FILE_EXTENSION" = "png" ]; then
    echo "image/png"
  elif [ "$FILE_EXTENSION" = "gif" ]; then
    echo "image/gif"
  elif [ "$FILE_EXTENSION" = "webp" ]; then
    echo "image/webp"
  elif [ "$FILE_EXTENSION" = "tif" ]; then
    echo "image/tif"
  elif [ "$FILE_EXTENSION" = "tiff" ]; then
    echo "image/tiff"
  elif [ "$FILE_EXTENSION" = "mp4" ]; then
    echo "video/mp4"
  elif [ "$FILE_EXTENSION" = "m3u8" ]; then
    echo "application/vnd.apple.mpegurl"
  elif [ "$FILE_EXTENSION" = "ts" ]; then
    echo "video/MP2T"
  elif [ "$FILE_EXTENSION" = "mp3" ]; then
	echo "audio/mp3"
  elif [ "$FILE_EXTENSION" = "txt" ]; then
    echo "text/plain"
  elif [ "$FILE_EXTENSION" = "html" ]; then
    echo "text/html"
  elif [ "$FILE_EXTENSION" = "css" ]; then
    echo "text/css"
  elif [ "$FILE_EXTENSION" = "js" ]; then
    echo "text/javascript"
  elif [ "$FILE_EXTENSION" = "csv" ]; then
    echo "application/msword"
  elif [ "$FILE_EXTENSION" = "doc" ]; then
    echo "application/vnd"
  elif [ "$FILE_EXTENSION" = "ppt" ]; then
    echo "application/vnd"
  elif [ "$FILE_EXTENSION" = "pdf" ]; then
    echo "application/pdf"
  elif [ "$FILE_EXTENSION" = "gz" ]; then
    echo "application/gzip"
  elif [ "$FILE_EXTENSION" = "zip" ]; then
    echo "application/zip"
  elif [ "$FILE_EXTENSION" = "rar" ]; then
    echo "application/x-rar-compressed"
  else
    echo "application/octet-stream"
  fi
}

HOST=$1
ACCESS_KEY=$2
SECRET_KEY=$3
BUCKET=$4
OBJECT_NAME=$5
FILE=$6
#HOST="127.0.0.1:8060"
#ACCESS_KEY="5LXV41WXDJY06J4E0ZEE"
#SECRET_KEY="VrEXTclX5uDhi0Op2wBFvUTghP03V96CMVsC4QiE"
#BUCKET="test"
#OBJECT_NAME="test/000.mp4"
#FILE="/Users/test/000.mp4"

RESOURCE="/${BUCKET}/${OBJECT_NAME}"
#echo "HOST: $HOST"
#echo "ACCESS_KEY: $ACCESS_KEY"
#echo "SECRET_KEY: $SECRET_KEY"
#echo "BUCKET: $BUCKET"
#echo "OBJECT_NAME: $OBJECT_NAME"
#echo "FILE: $FILE"
#echo "RESOURCE: $RESOURCE"

FILE_NAME="$(basename "$FILE")"
FILE_PREFIX="${FILE_NAME%.*}"
FILE_EXTENSION="${FILE_NAME##*.}"
CONTENT_TYPE="$(echo $(contentType))"
if [ "$(uname)" == "Darwin" ]; then
  FILE_SIZE=$(stat -f%z "$FILE")
else
  FILE_SIZE=$(stat -c%s "$FILE")
fi
#echo "FILE_NAME: $FILE_NAME"
#echo "FILE_PREFIX: $FILE_PREFIX"
#echo "FILE_EXTENSION: $FILE_EXTENSION"
#echo "CONTENT_TYPE: $CONTENT_TYPE"
#echo "FILE_SIZE: $FILE_SIZE"

ACL="x-amz-acl:public-read"
META_DATA="x-amz-meta-ukey:value"
CURRENT_TIME=`TZ=GMT LANG=en_US date "+%a, %d %b %Y %H:%M:%S GMT"`
stringToSign="PUT\n\n${CONTENT_TYPE}\n${CURRENT_TIME}\n${ACL}\n${META_DATA}\n${RESOURCE}"
signature=`echo -en ${stringToSign} | openssl sha1 -hmac ${SECRET_KEY} -binary | base64`
#echo "ACL: $ACL"
#echo "META_DATA: $META_DATA"
#echo "CURRENT_TIME: $CURRENT_TIME"
#echo "stringToSign: $stringToSign"
#echo "signature: $signature"

#curl -s -v -# \
#-X PUT "http://${HOST}${RESOURCE}" \
#-H "Host: ${HOST}" \
#-H "Date: ${CURRENT_TIME}" \
#-H "${ACL} " \
#-H "${META_DATA} " \
#-H "Content-Type: ${CONTENT_TYPE} " \
#-H "Content-Length: ${FILE_SIZE}" \
#-H "Authorization: AWS ${ACCESS_KEY}:${signature}" \
#-T "${FILE}"

code=$(curl -s -o /dev/null -w %{http_code} \
-X PUT "http://${HOST}${RESOURCE}" \
-H "Host: ${HOST}" \
-H "Date: ${CURRENT_TIME}" \
-H "${ACL} " \
-H "${META_DATA} " \
-H "Content-Type: ${CONTENT_TYPE} " \
-H "Content-Length: ${FILE_SIZE}" \
-H "Authorization: AWS ${ACCESS_KEY}:${signature}" \
-T "${FILE}"
)
#echo "curl put http_code: $code"
if [ "$code" = 200 ]; then
  exit 0
else
  exit 1
fi

```

S3-Ceph下载脚本
```
#!/bin/bash

# 根据文件扩展名获取Content-Type(Mime-Type)
contentType() {
  if [ "$FILE_EXTENSION" = "bmp" ]; then
    echo "image/bmp"
  elif [ "$FILE_EXTENSION" = "jpg" ]; then
    echo "image/jpeg"
  elif [ "$FILE_EXTENSION" = "jpeg" ]; then
    echo "image/jpeg"
  elif [ "$FILE_EXTENSION" = "png" ]; then
    echo "image/png"
  elif [ "$FILE_EXTENSION" = "gif" ]; then
    echo "image/gif"
  elif [ "$FILE_EXTENSION" = "webp" ]; then
    echo "image/webp"
  elif [ "$FILE_EXTENSION" = "tif" ]; then
    echo "image/tif"
  elif [ "$FILE_EXTENSION" = "tiff" ]; then
    echo "image/tiff"
  elif [ "$FILE_EXTENSION" = "mp4" ]; then
    echo "video/mp4"
  elif [ "$FILE_EXTENSION" = "m3u8" ]; then
    echo "application/vnd.apple.mpegurl"
  elif [ "$FILE_EXTENSION" = "ts" ]; then
    echo "video/MP2T"
  elif [ "$FILE_EXTENSION" = "mp3" ]; then
	echo "audio/mp3"
  elif [ "$FILE_EXTENSION" = "txt" ]; then
    echo "text/plain"
  elif [ "$FILE_EXTENSION" = "html" ]; then
    echo "text/html"
  elif [ "$FILE_EXTENSION" = "css" ]; then
    echo "text/css"
  elif [ "$FILE_EXTENSION" = "js" ]; then
    echo "text/javascript"
  elif [ "$FILE_EXTENSION" = "csv" ]; then
    echo "application/msword"
  elif [ "$FILE_EXTENSION" = "doc" ]; then
    echo "application/vnd"
  elif [ "$FILE_EXTENSION" = "ppt" ]; then
    echo "application/vnd"
  elif [ "$FILE_EXTENSION" = "pdf" ]; then
    echo "application/pdf"
  elif [ "$FILE_EXTENSION" = "gz" ]; then
    echo "application/gzip"
  elif [ "$FILE_EXTENSION" = "zip" ]; then
    echo "application/zip"
  elif [ "$FILE_EXTENSION" = "rar" ]; then
    echo "application/x-rar-compressed"
  else
    echo "application/octet-stream"
  fi
}

HOST=$1
ACCESS_KEY=$2
SECRET_KEY=$3
BUCKET=$4
OBJECT_NAME=$5
FILE=$6
#HOST="127.0.0.1:8060"
#ACCESS_KEY="5LXV41WXDJY06J4E0ZEE"
#SECRET_KEY="VrEXTclX5uDhi0Op2wBFvUTghP03V96CMVsC4QiE"
#BUCKET="test"
#OBJECT_NAME="test/000.mp4"
#FILE="/Users/test/000.mp4"

RESOURCE="/${BUCKET}/${OBJECT_NAME}"
#echo "HOST: $HOST"
#echo "ACCESS_KEY: $ACCESS_KEY"
#echo "SECRET_KEY: $SECRET_KEY"
#echo "BUCKET: $BUCKET"
#echo "OBJECT_NAME: $OBJECT_NAME"
#echo "FILE: $FILE"
#echo "RESOURCE: $RESOURCE"

FILE_PATH="$(dirname "$FILE")"
FILE_NAME="$(basename "$FILE")"
FILE_PREFIX="${FILE_NAME%.*}"
FILE_EXTENSION="${FILE_NAME##*.}"
CONTENT_TYPE="$(echo $(contentType))"
#echo "FILE_PATH: $FILE_PATH"
#echo "FILE_NAME: $FILE_NAME"
#echo "FILE_PREFIX: $FILE_PREFIX"
#echo "FILE_EXTENSION: $FILE_EXTENSION"
#echo "CONTENT_TYPE: $CONTENT_TYPE"

mkdir -p "$FILE_PATH"

ACL="x-amz-acl:public-read"
META_DATA="x-amz-meta-ukey:value"
CURRENT_TIME=`TZ=GMT LANG=en_US date "+%a, %d %b %Y %H:%M:%S GMT"`
stringToSign="GET\n\n${CONTENT_TYPE}\n${CURRENT_TIME}\n${ACL}\n${META_DATA}\n${RESOURCE}"
signature=`echo -en ${stringToSign} | openssl sha1 -hmac ${SECRET_KEY} -binary | base64`
#echo "ACL: $ACL"
#echo "META_DATA: $META_DATA"
#echo "CURRENT_TIME: $CURRENT_TIME"
#echo "stringToSign: $stringToSign"
#echo "signature: $signature"

#curl -s -v -# \
#-X GET "http://${HOST}${RESOURCE}" \
#-H "Host: ${HOST}" \
#-H "Date: ${CURRENT_TIME}" \
#-H "${ACL} " \
#-H "${META_DATA} " \
#-H "Content-Type: ${CONTENT_TYPE} " \
#-H "Authorization: AWS ${ACCESS_KEY}:${signature}" \
#-o "${FILE}"

code=$(curl -s -w %{http_code} \
-X GET "http://${HOST}${RESOURCE}" \
-H "Host: ${HOST}" \
-H "Date: ${CURRENT_TIME}" \
-H "${ACL} " \
-H "${META_DATA} " \
-H "Content-Type: ${CONTENT_TYPE} " \
-H "Authorization: AWS ${ACCESS_KEY}:${signature}" \
-o "${FILE}"
)
#echo "curl get http_code: $code"
if [ "$code" = 200 ]; then
  exit 0
else
  exit 1
fi

```



