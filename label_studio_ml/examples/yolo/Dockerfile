# Use the Jetson-compatible base image for TensorRT and CUDA
FROM nvcr.io/nvidia/l4t-tensorrt:r8.6.2-devel

ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /app

# Install basic dependencies (label studio)
RUN --mount=type=cache,target="/var/cache/apt",sharing=locked \
    --mount=type=cache,target="/var/lib/apt/lists",sharing=locked \
    apt-get -y update \
    && apt-get install -y git \
    && apt-get install -y wget \
    && apt-get install -y g++ freeglut3-dev build-essential libx11-dev \
    libxmu-dev libxi-dev libglu1-mesa libglu1-mesa-dev libfreeimage-dev \
    && apt-get -y install ffmpeg libsm6 libxext6 libffi-dev python3-dev python3-pip gcc


# Update package lists (jetson docker)
RUN apt-get update && \
    apt-get install -y python3-smbus&& \
    apt-get clean\
    && rm -rf /var/lib/apt/lists/* 

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
PYTHONDONTWRITEBYTECODE=1 \
PIP_CACHE_DIR=/.cache1 \
PORT=9090 \
WORKERS=2 \
THREADS=4

#
# Compile and install opencv with cuda enabled - takes a hour & 15 min for this one piece
#

RUN echo "/usr/local/cuda/lib64" >> /etc/ld.so.conf.d/nvidia-tegra.conf && ldconfig

RUN apt-get update && apt-get dist-upgrade -y --autoremove \
	&& apt-get install -y --no-install-recommends libswresample-dev libdc1394-dev cmake \
	libjpeg-dev libjpeg8-dev libjpeg-turbo8-dev libpng-dev libtiff-dev libglew-dev \
	libavcodec-dev libavformat-dev libswscale-dev libgtk2.0-dev libgtk-3-dev libcanberra-gtk* \
	libxvidcore-dev libx264-dev libtbb-dev libxine2-dev libv4l-dev v4l-utils qv4l2 \
	libtesseract-dev libpostproc-dev libvorbis-dev libfaac-dev libmp3lame-dev libtheora-dev \
	libopencore-amrnb-dev libopencore-amrwb-dev libopenblas-dev libatlas-base-dev libblas-dev \
	liblapack-dev liblapacke-dev libeigen3-dev gfortran libhdf5-dev libprotobuf-dev protobuf-compiler \
	libgoogle-glog-dev libgflags-dev build-essential libcanberra-gtk3-module \ 
	libgstreamer-plugins-base1.0-dev libgstreamer-plugins-good1.0-dev libgstreamer1.0-dev libtbb2 pkg-config \
	python3-dev python3-matplotlib zlib1g-dev && rm -rf /var/lib/apt/lists/*

ENV CUDA_HOME=/usr/local/cuda
ENV PATH=${PATH}:${CUDA_HOME}/bin
ENV LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:${CUDA_HOME}/lib64

WORKDIR /tmp
RUN  git clone --depth 1 --branch "4.10.0" https://github.com/opencv/opencv.git
RUN  git clone --depth 1 --branch "4.10.0" https://github.com/opencv/opencv_contrib.git

WORKDIR /tmp/opencv/build
RUN cmake -D BUILD_EXAMPLES=OFF \
    	-D BUILD_opencv_python2=ON \
    	-D BUILD_opencv_python3=ON \
    	-D CMAKE_BUILD_TYPE=RELEASE \
    	-D CMAKE_INSTALL_PREFIX=/usr/local \
    	-D CUDA_ARCH_BIN=8.7 \
    	-D CUDA_ARCH_PTX="" \
    	-D CUDA_FAST_MATH=ON \
    	-D EIGEN_INCLUDE_PATH=/usr/include/eigen3 \
    	-D CUDNN_VERSION=9.0 \
    	-D ENABLE_NEON=ON \
    	-D OPENCV_DNN_CUDA=ON \
    	-D OPENCV_ENABLE_NONFREE=ON \
    	-D OPENCV_EXTRA_MODULES_PATH=/tmp/opencv_contrib/modules \
    	-D OPENCV_GENERATE_PKGCONFIG=ON \
    	-D WITH_CUBLAS=ON \
    	-D WITH_CUDA=ON \
    	-D WITH_CUDNN=ON \
    	-D WITH_GSTREAMER=ON \
    	-D WITH_LIBV4L=ON \
    	-D WITH_OPENGL=ON \
    	.. && \
    make -j 6 && \
    make install && \
    rm -rf /tmp/* 


#
# Install awscli
#

RUN apt-get update && apt-get install -y curl unzip
WORKDIR /tmp
RUN 	curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip" \
    	&& unzip awscliv2.zip \
    	&& cd aws \
    	&& ./install \
    	&& cd .. \
    	&& rm -rf awscliv2.zip


#
# Install pytorch
#  

RUN apt install -y openmpi-bin
RUN apt-get -y update && apt-get -y install libopenblas-dev
RUN wget -O torch-2.3.0-cp310-cp310-linux_aarch64.whl https://nvidia.box.com/shared/static/mp164asf3sceb570wvjsrezk1p4ftj8t.whl \
	&& wget -O torchvision-0.18.0-cp310-cp310-linux_aarch64.whl https://nvidia.box.com/shared/static/xpr06qe6ql3l6rj22cu3c45tz1wzi36p.whl
RUN	pip install --upgrade pip
RUN	pip install numpy==1.26.1
RUN	pip install --no-cache torch-2.3.0-cp310-cp310-linux_aarch64.whl
RUN	pip install --no-cache torchvision-0.18.0-cp310-cp310-linux_aarch64.whl
RUN	rm torch-2.3.0-cp310-cp310-linux_aarch64.whl
RUN	rm torchvision-0.18.0-cp310-cp310-linux_aarch64.whl    

#
# Install required packages - put pip_requirements in Projects/MV folder
#

RUN apt-get update && apt-get install -y dbus
RUN apt-get install -y libusb-1.0-0-dev
RUN apt-get install -y supervisor

COPY pip_requirements.txt pip_requirements.txt
RUN pip install -r pip_requirements.txt   

# Copy your application files - this will be the detect script and it must be in Projects/MV folder
COPY . /app
COPY utils /app/utils
# COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

WORKDIR /app

# Install YOLOv9 dependencies and Label Studio backend requirements
# install base requirements
COPY requirements-base.txt .
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    pip install -r requirements-base.txt

# install model requirements
COPY requirements.txt .
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    pip3 install -r requirements.txt

# install test requirements if needed
COPY requirements-test.txt .
# build only when TEST_ENV="true"
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    if [ "$TEST_ENV" = "true" ]; then \
      pip3 install -r requirements-test.txt; \
    fi

WORKDIR /app

COPY . ./

# Download the YOLO models
# RUN /bin/sh -c 'if [ ! -f /app/models/yolov8m.pt ]; then \
#     && yolo predict model=/app/models/yolov8n.pt source=/app/tests/car.jpg \
#     && yolo predict model=/app/models/yolov8n-cls.pt source=/app/tests/car.jpg \
#     && yolo predict model=/app/models/yolov8n-seg.pt source=/app/tests/car.jpg; \
#     fi'

ENV PYTHONPATH=/app

# Expose necessary ports
EXPOSE 9090

# Start the Label Studio ML backend service
CMD ["/app/start.sh"]