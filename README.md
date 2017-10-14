# Building-Tensorflow1.3.1-with-CUDA9-CUDNN7-on-UBUNTU-LINUX

（17.10.14更新)
Tensorflow1.4.0rc0 が出たのと、GLIBC で CUDA と相性悪い件、とりあえず解消されたので

(17.10.1更新）
cuDNN 7.0.3 登場したのと、bazel が 0.6.0 になったけど動かないので 0.5.4 を入れ直す必要があることを追記。

(17.9.29更新)
CUDA9.0GA がリリース＋Tensorflow1.3.1 が公開されたので。

Ubuntu Linux 上で、CUDA　9.0　+　cuDNN　7.0 でtensorflow　1.3.1 が構築できたよ。

Tensorflow 1.3.0 --> Tensorflow 1.3.1 ではロジックに手は加わってない。そのかわり

・　「これが一番デカい」コンパイル時に必要な依存ライブラリのダウンロードが(SHA256ハッシュ値が古くなったせいで？)失敗するようになってた不具合を修正

・　Python3.6 にも正式対応

・　CUDA 8.0, cudnn 6.0 でないと対応しない (CUDA 7.5, cudnn 5.X はダメらしい試してないけど)

ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー

環境

GPU: NVidia Geforce GTX1050/1050Ti/1060/1070/1080/1080Ti

OS:  UBUNTU 16.04 , 17.04 　, 17.10 (glibc+CUDA8/9 　の不具合は glibc 2.26-0ubuntu2 で回避されました)


 以下は構築の手順
 

必要なものドライバ等


1. GTX10XX 用ドライバ

sudo add-apt-repository ppa:graphics-drivers/ppa

2． CUDA 9.0 

https://developer.nvidia.com/cuda-downloads

3. cuDNN 7.0.3

https://developer.nvidia.com/cudnn

1〜3のインストールやりかたはここらへんに

http://qiita.com/conta_/items/d639ef0068c9b7a0cd12


CUDA,cuDNN は以下レポジトリから取得すればよい。

deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1704/x86_64 /

deb http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /

（署名）

sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1704/x86_64/7fa2af80.pub

実際のURLは、

http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1704/x86_64/

http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64



※PATH等　環境変数はお忘れ無く


TENSORFLOW


1．. Tensorflow 1.3.1

https://github.com/tensorflow/tensorflow/releases/tag/v1.3.1

2. CUDA9.0 用パッチ　（TF 1.4.0 以降では不要）

https://github.com/tensorflow/tensorflow/issues/12474

※　パッチは2つリンクがあるけど、下の方。具体的にはこれ。

wget https://github.com/tensorflow/tensorflow/files/1253794/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt

patch -p1 < ~/Downloads/0002-TF-1.3-CUDA-9.0-and-cuDNN-7.0-support.patch.txt
 

ここらへんの修正も必要なもかもしれないが未確認

https://github.com/tensorflow/tensorflow/pull/12502


3． Bazel（Googleのビルド用t−ル）を入れる。

https://bazel.build/

※Javaを入れる必要があるが、Java9は不可。Java8で。

2017.9.28 に bazel 0.6.0 が登場したけど、ビルド不可 (bzlファイルが対応してない）ので、 bazel 0.5.4 にする必要がある

以下リンクからダウンロード & dpkg すればよい。

https://github.com/bazelbuild/bazel/releases/tag/0.5.4


4． ./configure をかける

・　 CUDA NVCC のバックエンドで使うGCCは、4.X でも 6 でも動く。ただし 7 ではビルド不可。

・ 　cuDNN のバージョンは "7.0.3" と BuildNo まで入れること


5. （冒頭のとおり、ライブラリが正常にダウンロードできるようになったのでこの箇所は削除）

参考：（ちと古いが）

https://hinaser.github.io/Machine-Learning/deeplearning-by-tensorflow-with-gpu.html


6．実際にビルドに入る。

GPUによる処理メインでも、CPU の SSE やらAVXは使えるに越したことはないと思うので、そのためのお忘れなく。

とはいえ、例えば、SkyLake (6X00/7X00)なら　-maarch=skylake にすればよい。（mavx512 がビルドできない、という書き込みを見たけど、XEON　も　core-X にもアクセスできないので真偽は不明）。

詳細は、gcc ドキュメントを参照

https://gcc.gnu.org/onlinedocs/gcc-6.1.0/gcc/x86-Options.html#x86-Options

とりあえず、自分のビルドコマンド例 (-ffast-math は1.4以降では動かないっぽい・)

bazel build -c opt   --config=cuda --copt=-march=skylake --copt=-mfpmath=both --copt=-O3 //tensorflow/tools/pip_package:build_pip_package
 
7．　（TF 1.4.0 以降では不要）
ただしこの時点では(CUDA9.0では）エラーが出るので、別のパッチ（上記ISSUEにも記載あり。Eigenまわりのパッチらしい）をダウンロード＆パッチ当てしてビルド再実行。

wget https://storage.googleapis.com/tf-performance/public/cuda9rc_patch/eigen.f3a22f35b044.cuda9.diff

cd -P bazel-out/../../../external/eigen_archive
 
patch -p1 < ~/Downloads/eigen.f3a22f35b044.cuda9.diff
    
元のフォルダに戻って、bazel を再開⇒今度はうまくいくはず。

で、最後に

bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg

でビルド完了。
