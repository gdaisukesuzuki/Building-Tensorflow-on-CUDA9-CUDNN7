# Building-Tensorflow1.3.1-with-CUDA9-CUDNN7-on-UBUNTU-LINUX

(17.9.28更新)
CUDA9.0GA がリリース＋Tensorflow1.3.1 が公開されたので。

UBUNTU Linux 上で、CUDA　9.0　+　CUDNN　7.0 でtensorflow　1.3.1 が構築できたよ。

Tensorflow 1.3.0 --> Tensorflow 1.3.1 ではロジックに手は加わってない。そのかわり

・　Python3.6 にも正式対応

・　ライブラリダウンロードが失敗するようになってた不具合を修正

・CUDA 8.0, cudnn 6.0 でないと対応しない (CUDA 7.5, cudnn 5.X はダメらしい試してないけど)



環境

GPU: NVidia Geforce GTX1050/1050Ti/1060/1070/1080/1080Ti

OS:  UBUNTU 16.04 , 17.04 　, 17.10 (17.10 はビルドできるけど注意が必要 at 2017.9.14 時点）

　ubuntu 17.10beta ではそのままでは　ビルド不可（CUDA8.0もビルドNG）。
 
 
 具体的には、GNU LIBC 2.26 以上でコンパイル不可。NVCC　が　”__float128” をサポートしてないのが原因。
 
 "CUDA C PROGRAMMING GUIDE" のCUDA 9.0 にも書かれてるので（F.3.1 Host Restrictions）、多分来年のCUDA10.0 まではこの状態。
 
 http://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html

 以下は、CUDA8.0 の議論なのだが、CUDA9.0 でも同様。Ubuntu17.04 のサポート期限は来年1月までなので、そこまでには解消されると思われるが, NVIDIA にやる気を出して貰うしかない状態。
 
 
 https://devtalk.nvidia.com/default/topic/1023776/-request-add-nvcc-compatibility-with-glibc-2-26/
 

*回避策

 なので、ちょー邪道な方法だが、glibc2.26 で、__float128 を宣言を無効にするしかない。
 
 具体的には
 
 /usr/include/x86_64-linux-gnu/bits/floatn.h
 
 を、やっつけだが、以下パッチ(NVCCを通した場合は、 __float128 を無効に設定する）適用して、
 
 https://raw.githubusercontent.com/gdaisukesuzuki/Building-Tensorflow-on-CUDA9RC-CUDNN7/master/floatn.h-patch
 
 回避はできた。。ubuntu とかにマージリクエストを出したけど、通るかは自信がない。CUDA のことだしね。
 
 ちなみに、archlinux ではすでに議論にはあがっていた。

https://www.reddit.com/r/archlinux/comments/6zrmn1/torch_on_arch/

　※　余談だが、このパッチは、　CUPY 2.0 を UBUNTU17.10 でビルドする場合にも必要（　nvcc.profile でバックエンドに使うコンパイラを GCC6（あるいはそれより前）に変更するだけではダメ）。


 以下は構築の手順
 

必要なものドライバ等


1. GTX10XX 用ドライバ

sudo add-apt-repository ppa:graphics-drivers/ppa

2． CUDA 9.0 

https://developer.nvidia.com/cuda-downloads

3. CUDNN 7.0.2

https://developer.nvidia.com/cudnn

1〜3のインストールやりかたはここらへんに

http://qiita.com/conta_/items/d639ef0068c9b7a0cd12

※PATH等　環境変数はお忘れ無く


TENSORFLOW


1．. Tensorflow 1.3.1

https://github.com/tensorflow/tensorflow/releases/tag/v1.3.1

2. CUDA9.0 用パッチ

https://github.com/tensorflow/tensorflow/issues/12474

※　パッチは2つリンクがあるけど、下の方。具体的にはこれ。

wget https://github.com/tensorflow/tensorflow/files/1253794/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt

patch -p1 < ~/Downloads/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt
 

ここらへんの修正も必要なもかもしれないが未確認

https://github.com/tensorflow/tensorflow/pull/12502

3． Bazel（Googleのビルド用t−ル）を入れる。

https://bazel.build/

※Javaを入れる必要があるが、Java9は不可。Java8で。

4． ./configure をかける

・　 CUDA で使うGCCは、4.X でも 6 でも動く。ただし 7 ではビルド不可。

・ 　CUDNN のバージョンは 7.0.2 とBuildNo まで入れること


5. （冒頭のとおり、ライブラリが正常にダウンロードできるようになったのでこの箇所は削除）

参考：（ちと古いが）

https://hinaser.github.io/Machine-Learning/deeplearning-by-tensorflow-with-gpu.html


6．実際にビルドに入る。

GPUによる処理メインでも、CPU の SSE やらAVXは使えるに越したことはないと思うので、そのためのお忘れなく。

とはいえ、例えば、SkyLake (6X00/7X00)なら　-maarch=skylake にすればよい。（mavx512 がビルドできない、という書き込みを見たけど、XEON　も　core-X にもアクセスできないので真偽は不明）。

詳細は、gcc ドキュメントを参照

https://gcc.gnu.org/onlinedocs/gcc-6.1.0/gcc/x86-Options.html#x86-Options

とりあえず、自分のビルドコマンド例

bazel build -c opt   --config=cuda --copt=-march=skylake --copt=-mfpmath=both --copt=-ffast-math --copt=-O3 //tensorflow/tools/pip_package:build_pip_package
 
7．ただしこの時点では(CUDA9.0では）エラーが出るので、別のパッチ（上記ISSUEにも記載あり。Eigenまわりのパッチらしい）をダウンロード＆パッチ当てしてビルド再実行。

wget https://storage.googleapis.com/tf-performance/public/cuda9rc_patch/eigen.f3a22f35b044.cuda9.diff

cd -P bazel-out/../../../external/eigen_archive
 
patch -p1 < ~/Downloads/eigen.f3a22f35b044.cuda9.diff
    
元のフォルダに戻って、bazel を再開⇒今度はうまくいくはず。

で、最後に

bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg

でビルド完了。
