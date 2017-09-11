# Building-Tensorflow-on-CUDA9RC-CUDNN7
CUDA9.0RC+CUDNN7.0 でtensorflow が構築できたよ。



環境
GPU: GTX 10XX
OS:  UBUNTU 16.04 , 17.04 　（17.10 は2017.9.10 現在ダメ）

　17.10beta は行けるかと思いましたが、パッケージ更新でビルド不可に。
 
 具体的には、GNU LIBC 2.26 以上でコンパイル不可（typedef のパースがおかしい？）なのが原因。
 
 以下は、CUDA8.0 の議論なのだが、CUDA9.0 でも同様。Ubuntu17.04 のサポート期限は来年1月までなので、そこまでには解消されると思われるが, NVIDIA にやる気を出して貰うしかない状態。
 
 https://devtalk.nvidia.com/default/topic/1023776/-request-add-nvcc-compatibility-with-glibc-2-26/
 


必要なものドライバ等


1. GTX10XX 用ドライバ

sudo add-apt-repository ppa:graphics-drivers/ppa

2． CUDA 9.0 RC

https://developer.nvidia.com/cuda-release-candidate-download

3. CUDNN7.0.1 ⇒ CUDNN7.0.2

https://developer.nvidia.com/cudnn

1〜3のインストールやりかたはここらへんに

http://qiita.com/conta_/items/d639ef0068c9b7a0cd12
※PATH等　環境変数はお忘れ無く


TENSORFLOW


1．. Tensorflow1.3.0

https://github.com/tensorflow/tensorflow/releases/tag/v1.3.0

2. CUDA9.0 用パッチ

https://github.com/tensorflow/tensorflow/issues/12474
※　パッチは2つリンクがあるけど、下の方。

ここらへんの修正も必要なもかもしれないが未確認
https://github.com/tensorflow/tensorflow/pull/12502


3． ./configure をかける

・　 GCCは、4.X でも 6 でも動く。ただし 7 ではビルド不可。
・ 　CUDNN のバージョンは 7.0.2 とBuildNo まで入れること

ここから bazel でビルドすればよいだけだが、 CUDA のヘッダファイルが悪さをしてるので、応急処置で

＄CUDA_HOME/include/crt/common_functions.h
の

__CUDACC_VER__

に関する記述を変更した


#define __CUDACC_VER__  ((__CUDACC_VER_MAIN__)*10000+(__CUDACC_VER_MINOR__)*100+(__CUDACC_VER_BUILD__))

4．あとはbazel を実行すれば構築できるが、CPU の SSE やらAVXは使えるに越したことはないと思うので、お忘れ無く。（mavx512 がビルドできない、という書き込みを見たけど、XEONも core-X にもアクセスできないので真偽は不明）

とりあえず、自分のビルドコマンド例

bazel build -c opt --config=cuda --copt=-mavx --copt=-mavx2 --copt=-mmmx --copt=-mfma --copt=-msse3 --copt=-msse4.1 --copt=-msse4.2 --copt=-mfpmath=both  --copt=-ffast-math   //tensorflow/tools/pip_package:build_pip_package 

bazel の使った一連のビルド操作は、この記事が参考になるかと。

https://hinaser.github.io/Machine-Learning/deeplearning-by-tensorflow-with-gpu.html


