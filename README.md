FFmpeg scale_cuda 10bit problem
---

# <a name="about"></a>Description

When scaling a 10bit video using `scale_cuda` filter (witch uses pixel format `AV_PIX_FMT_P010LE`), the output video gets distorted.

I suppose it has something to do with the differences in processing between cuda_sdk and ffnvcodec with cuda_nvcc (the problem appears after this commit [https://github.com/FFmpeg/FFmpeg/commit/2544c7ea67ca9521c5de36396bc9ac7058223742](https://github.com/FFmpeg/FFmpeg/commit/2544c7ea67ca9521c5de36396bc9ac7058223742)).

**Demonstration of the problem:**
![https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/master/screenshots/problem.jpg](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/master/screenshots/problem.jpg)

**What it should look like:**
![https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/master/screenshots/fix.jpg](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/master/screenshots/fix.jpg)

This repository is intended to prove that such problem exists and to show, how to solve it.

# <a name="preparing"></a>Preparation steps

In order to reproduce the problem, you should have:

1. Linux environment;
2. Nvidia graphic card with cuvid/nvenc support and latest drivers installed (I tested on GTX 1080, GTX 1050 and 430.09/418.56 drivers);
3. Docker installed (for building and running FFmpeg). Using docker is not mandatory, but recommended, as it simplifies the process of building;
4. [Nvidia docker runtime](https://github.com/NVIDIA/nvidia-docker/) installed (if using docker).

Building the image:

`docker build images -f images/Dockerfile -t ffmpeg-scale-10bit-test`

# <a name="running"></a>Reproducing the problem

I prepared a sample video with yuv420p10le pixel format [samples/input.ts](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/master/samples/input.ts).

Let's try to scale it down to HD-ready:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-scale-10bit-test ffmpeg -y -hwaccel cuvid -c:v hevc_cuvid -i /samples/input.ts -map 0:0 -c:v hevc_nvenc -vf scale_cuda=1280:720 /samples/problem.ts`

After the ffmpeg process completes try playing the output file `samples/problem.ts`.
You will immediately see that the picture is distorted.

# <a name="running"></a>Fixing the problem

To solve the problem we should not divide the input frame planes' `linesize`s by 2 ([1](https://github.com/FFmpeg/FFmpeg/blob/dcc999819dda578a4d8e52c6d17bf55d0073783d/libavfilter/vf_scale_cuda.c#L426), [2](https://github.com/FFmpeg/FFmpeg/blob/dcc999819dda578a4d8e52c6d17bf55d0073783d/libavfilter/vf_scale_cuda.c#L430)) and leave them as they are.

I prepared a patch [images/fix.patch](https://github.com/Svechnikov/ffmpeg-scale-cuda-10bit-problem/blob/master/images/fix.patch). You can test the patch using a special docker-image:

`docker build images -f images/Dockerfile.fixed -t ffmpeg-scale-10bit-test`

If you run the commands again, the output video will be correct.
