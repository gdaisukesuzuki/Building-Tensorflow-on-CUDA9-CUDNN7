# Building-Tensorflow-on-CUDA9RC-CUDNN7
CUDA9.0RC+CUDNN7.0 でtensorflow が構築できたよ。



環境
GPU: GTX 10XX
OS:  UBUNTU 16.04 , 17.04 　（17.10 はできるけど注意が必要 at 2017.9.14 時点）

　17.10beta パッケージ更新でビルド不可に。
 
 具体的には、GNU LIBC 2.26 以上でコンパイル不可。（現時点ではproposedパッケージだが本番にはすぐ反映されるかと）。端的には、　　NVCC　が　__float128 をサポートしてないのが原因。"CUDA C PROGRAMMING GUIDE" のCUDA 9.0 にも書かれてるのでしょうがない（F.3.1 Host Restrictions）。
 
 http://docs.nvidia.com/cuda-rc/cuda-c-programming-guide/index.html

 以下は、CUDA8.0 の議論なのだが、CUDA9.0 でも同様。Ubuntu17.04 のサポート期限は来年1月までなので、そこまでには解消されると思われるが, NVIDIA にやる気を出して貰うしかない状態。
 
 https://devtalk.nvidia.com/default/topic/1023776/-request-add-nvcc-compatibility-with-glibc-2-26/
 

*回避策

 なので、ちょー邪道な方法だが、glibc2.26 で、__float128 を宣言を無効にするしかない。
 具体的には
 
 /usr/include/x86_64-linux-gnu/bits/floatn.h
 
 を、やっつけだが、以下パッチ適用して、。
 
 https://raw.githubusercontent.com/gdaisukesuzuki/Building-Tensorflow-on-CUDA9RC-CUDNN7/master/floatn.h-patch
 
 ごまかし回避はできた。(NVCCを通した場合は、 __float128 を無効に設定する）。ubuntu とかにマージリクエストを出そうかと思うけど、通るかは自信がない。CUDA のことだしね。
 ちなみに、archlinux ではすでに議論にはあがっていた。

https://www.reddit.com/r/archlinux/comments/6zrmn1/torch_on_arch/


 以下は構築の手順
 

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
※　パッチは2つリンクがあるけど、下の方。具体的には

wget https://github.com/tensorflow/tensorflow/files/1253794/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt

patch -p1 < ~/Downloads/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt
 

ここらへんの修正も必要なもかもしれないが未確認
https://github.com/tensorflow/tensorflow/pull/12502


3． ./configure をかける

・　 GCCは、4.X でも 6 でも動く。ただし 7 ではビルド不可。
・ 　CUDNN のバージョンは 7.0.2 とBuildNo まで入れること




4．あとはbazel を実行すれば構築できるが、CPU の SSE やらAVXは使えるに越したことはないと思うので、お忘れ無く。（mavx512 がビルドできない、という書き込みを見たけど、XEONも core-X にもアクセスできないので真偽は不明）

とりあえず、自分のビルドコマンド例

bazel build -c opt --config=cuda --copt=-mavx --copt=-mavx2 --copt=-mmmx --copt=-mfma --copt=-msse3 --copt=-msse4.1 --copt=-msse4.2 --copt=-mfpmath=both  --copt=-ffast-math   //tensorflow/tools/pip_package:build_pip_package 

5．ただしこの時点ではエラーが出るので、別のパッチ（上記ISSUEにも記載あり）をダウンロード＆パッチ宛てして再度実行らしい。

wget https://storage.googleapis.com/tf-performance/public/cuda9rc_patch/eigen.f3a22f35b044.cuda9.diff

cd -P bazel-out/../../../external/eigen_archive
 
patch -p1 < ~/Downloads/eigen.f3a22f35b044.cuda9.diff
    
元のフォルダに戻って、bazel を再開⇒今度はうまくいくはず。

bazel の使った一連のビルド操作は、この記事が参考になるかと。

https://hinaser.github.io/Machine-Learning/deeplearning-by-tensorflow-with-gpu.html

