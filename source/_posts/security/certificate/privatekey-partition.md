---
title: 【安全贴士】私钥分片和管道口令读取
date: 2020-05-28 11:15:17 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

通过一个私钥分片和管道口令读取的脚步来自动签发数字证书
<!--more-->

## 私钥文件分片.sh

```bash
#!/bin/bash
# 私钥文件分片存储
# @author XiongNeng
# @since 2019/05/06

WORK_DIR="/home/ca/work"
key1=certs/B1BF8FF1C636C6FA.crt
key2=csr/server0.csr
key3=private/ca0.key
KEY_FILE="ca3.key"
SPLIT_PREFIX="ca3_"

split_key() {
	cd $WORK_DIR
	/bin/split $KEY_FILE -l 10 $SPLIT_PREFIX
	mv "${SPLIT_PREFIX}aa" $key1
	mv "${SPLIT_PREFIX}ab" $key2
	mv "${SPLIT_PREFIX}ac" $key3
}

merge_key() {
	cd $WORK_DIR
	cakey="/tmp/$(/bin/date +%s).key"
	/bin/cat $key1 $key2 $key3 > $cakey
	echo $cakey
}

if [[ "$#" < 1 || ("$1" != "split" && "$1" != "merge") ]]; then
    echo "Usage: ./split.sh split|merge"
    exit 1
fi

if [[ "$1" == "split" ]]; then
    split_key
else
    merge_key
fi
```

## 管道文件读取口令密钥.sh

```bash
#!/bin/bash
# 通过管道文件输入私钥密码，签发数字证书
# @author XiongNeng
# @since 2019/05/06

CRT="ca3.crt"
CSR="csr/server.csr"
OUT_CERT="certs/server3.crt"
WORK_DIR="/home/ca/work"
KEY_PASSWORD_FILE="${WORK_DIR}/keypass"
PROCESS_ACCOUNT="ca"

sign() {
	LOG_FILE="${WORK_DIR}/$(/bin/date +%s).log"
	# su - $PROCESS_ACCOUNT
	cd $WORK_DIR

	# 通过系统命令调用外部程序获取私钥密码
	KEY_PASSOWRD="123456"

	if [ "$KEY_PASSOWRD" = "" ];then
		echo "ERROR: cert key password is empty."
		return 1
	fi

	#创建管道文件
	# su -s /bin/sh $PROCESS_ACCOUNT -c "mkfifo ${WORK_DIR}/keypass"
	if [[ -p "${WORK_DIR}/keypass" ]]; then
	    echo "keypass pipe file exists, I will delete it"
	    rm -f "${WORK_DIR}/keypass"
	fi
	mkfifo ${WORK_DIR}/keypass
	if [ $? -ne 0 ];then
		echo "ERROR: mkfifo keypass failed."
		return 1
	fi

	#设置管道文件权限
	chown $PROCESS_ACCOUNT:$PROCESS_ACCOUNT ${WORK_DIR}/keypass
	if [ $? -ne 0 ]; then
		echo "ERROR: chown keypass failed."
		return 1
	fi

	chmod 600 ${WORK_DIR}/keypass
	if [ $? -ne 0 ]; then
		echo "ERROR: chmod keypass failed."
		return 1
	fi

	#以$PROCESS_ACCOUNT身份启动证书签名命令
	/bin/openssl x509 -req -days 3650 -CAcreateserial -CA $CRT -CAkey $1 -passin file:$KEY_PASSWORD_FILE -in $CSR -out $OUT_CERT &>${LOG_FILE} &

	#将密码写入管道文件
	echo "INFO: start echo key_password."
	echo $KEY_PASSOWRD >> ${WORK_DIR}/keypass 
	sleep 0.1

	#检查证书签发命令是否执行成功
	timeout=3
	num=0
	while [ $num -lt $timeout ]; do
		grep ":error:" ${LOG_FILE}
		if [ $? -eq 0 ]; then
			echo "Error: sign cert failed."
			return 1
		fi
		echo "sleep 1 seconds"
		sleep 1
		num=`expr $num + 1`
	done
	
	cat ${LOG_FILE}

	if [ $num -eq $timeout ]; then
		echo "Success: good luck."
	fi
	rm -f "${WORK_DIR}/keypass" ${LOG_FILE} $1
}

if [[ "$#" < 1 ]]; then
    echo "Usage: ./sign.sh /tmp/ca.key"
    exit 1
fi

sign $1
```