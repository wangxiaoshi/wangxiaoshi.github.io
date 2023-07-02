---
title: 分析 MOA 中 RandomTreeGenerator 的痛苦过程
date: 2020-01-23 12:32:11
tags: 
- MOA
- 论文
categories: 学习笔记
cover: https://raw.githubusercontent.com/wangxiaoshi/image-host-xiaoshi/master/blog_files/img/Trees.jpg
---

## 论文题目
我的论文题目是 “Analysis and Optimization for Hoeffding Tree”，主要用到一个 [java 的流机器学习框架：MOA](https://moa.cms.waikato.ac.nz/)，可以在[这里](https://github.com/waikato/moa)找到它的源码，我需要改变其中一个 learner “Hoeffding Tree”，让其获得更少的 Cache misses。为了能准确有效地评估结果，应该让其在对比测试中获得相同的输入 Stream，也就是将 `RandomTreeGenerator` 变成固定的一棵树。
****
## 分析结构
我尝试着找到moa具体在哪个阶段调用了 `RandomTreeGenerator`，但是迟迟没有头绪。我试着在 `generateRandomTreeNode` 中每次调用打印一次，可是奇怪的是：在设置中我选择了1000个 Node，却只调用了405次该方法。

## 测试流程
我准备了两种测试流程，针对不同的两种架构：
### 本地测试
在 IDEA 中完成 JAR 包的编译后（这也是我之后准备写进脚本的一步），我会在 moa 源码的根目录运行下面的 script：
```
cp "moa/classes/artifacts/moa_test/moa-test.jar" "moa-release-2019.05.0/lib/"
cd moa-release-2019.05.0/lib/
java -cp moa-test.jar -javaagent:sizeofag-1.0.4.jar moa.gui.GUI
```

### 远端测试
同样的，只不过需要先上传新的 `moa-test.jar` 到服务器上，再运行 MOA：
```
DIR=`pwd`
DIR="$DIR/moa/classes/artifacts/moa_test"

# Confirm that the file exists
printf "Locating moa-test.jar in $DIR...\n"
if [ ! -f "$DIR/moa-test.jar" ]
then
    printf "ERROR: moa-test.jar does not exist\n"
    exit 1
else
    printf "Found the expected jar package. Start uploading..\n"
fi

# Upload the file to remote server
scp "$DIR/moa-test.jar" "root@172.105.79.21:~/moa-release-2019.05.0/lib/"
#printf "Upload completed. Start running remote MOA script...\n"
printf "Upload completed. Due to unknown bugs please run the remote script manuelly.\n"

read -p "Press any key to start downloading the results..."

scp root@172.105.79.21:~/results/* ~/Documents/Uni/BA/results/
```
他会等待我们执行完远端的 MOA 程序再将运行结果下载回本地方便查看。

在 ssh 连接到服务器后，运行以下 `./run_moa.sh [both | origin | test]`  ：
```
printf "Results will be saved in:${DIR}\n"

DATE=`date +%Y-%m-%d-%H:%M:%S`

PERF_OPTS="cache-misses,cache-references,cpu-cycles,instructions,ref-cycles,alignment-faults,cpu-clock,emulation-faults,major-faults,minor-faults,page-faults,L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores,L1-icache-load-misses,branch-load-misses,branch-loads,dTLB-load-misses,dTLB-loads,dTLB-store-misses,dTLB-stores,iTLB-load-misses,iTLB-loads,branch-instructions,branch-misses,bus-cycles,cache-references,cpu-cycles,cycles-ct,cycles-t,instructions,l1d.replacement,l1d_pend_miss.fb_full,l1d_pend_miss.pending_cycles,l1d_pend_miss.pending_cycles_any,l2_demand_rqsts.wb_hit,l2_lines_in.all,l2_lines_in.e,l2_lines_in.i,l2_lines_in.s,l2_lines_out.demand_clean,l2_rqsts.all_code_rd,l2_rqsts.all_demand_data_rd,l2_rqsts.all_demand_miss,l2_rqsts.all_demand_references,l2_rqsts.all_pf,l2_rqsts.all_rfo,l2_rqsts.code_rd_hit,l2_rqsts.code_rd_miss,l2_rqsts.demand_data_rd_hit,l2_rqsts.demand_data_rd_miss,l2_rqsts.l2_pf_hit,l2_rqsts.l2_pf_miss,l2_rqsts.miss,l2_rqsts.references,l2_rqsts.rfo_hit,l2_rqsts.rfo_miss,l2_trans.all_pf,l2_trans.all_requests,l2_trans.all_pf,l2_trans.all_requests,l2_trans.code_rd,l2_trans.demand_data_rd,l2_trans.l1d_wb,l2_trans.l2_fill,l2_trans.l2_wb,l2_trans.rfo"

if [ "$1" = "both" ]; then
printf "Both jar package will be executed\n"
printf "Running with moa.jar ...\n"
perf stat -e ${PERF_OPTS} java -cp moa.jar -javaagent:sizeofag-1.0.4.jar moa.DoTask "EvaluatePrequential -l trees.HoeffdingTree -i 1000000 -f 10000" > ${DIR}dsresult${DATE}.csv
printf "Running with moa-test.jar ...\n"
perf stat -e ${PERF_OPTS} java -cp moa-test.jar -javaagent:sizeofag-1.0.4.jar moa.DoTask "EvaluatePrequential -l trees.HoeffdingTree -i 1000000 -f 10000" > ${DIR}dsresult${DATE}-test.csv

elif [ "$1" = "origin" ]; then
	printf "Runnning in original MOA ...\n"
	
	perf stat -e ${PERF_OPTS} java -cp moa.jar -javaagent:sizeofag-1.0.4.jar moa.DoTask "EvaluatePrequential -l trees.HoeffdingTree -i 1000000 -f 10000" > ${DIR}dsresult${DATE}.csv

elif [ "$1" = "test" ]; then
	printf "Running with moa-test.jar ...\n"
perf stat -e ${PERF_OPTS} java -cp moa-test.jar -javaagent:sizeofag-1.0.4.jar moa.DoTask "EvaluatePrequential -l trees.HoeffdingTree -i 1000000 -f 10000" > ${DIR}dsresult${DATE}-test.csv

else
	print [ERROR: Illegal parameter!];
	exit 1
fi
```
它会帮我们运行指定的 MOA 版本，并且输出 pref 的结果方便查看 cache-misses 的情况。结果会保存在 results 目录中。这时就可以在本地按任意键开始下载结果了。
***