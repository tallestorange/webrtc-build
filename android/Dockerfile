FROM ubuntu:18.04 AS builder
ARG WEBRTC_COMMIT
RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y git curl wget xz-utils python python3 python3-setuptools pkg-config rsync unzip
WORKDIR /root/
RUN git clone https://chromium.googlesource.com/chromium/tools/depot_tools
ENV PATH /root/depot_tools/:$PATH
ARG PACKAGE_NAME=android
ENV SOURCE_DIR "/root/_source/$PACKAGE_NAME"
ENV BUILD_DIR "/root/_build/$PACKAGE_NAME"
ENV PACKAGE_DIR "/root/_package/$PACKAGE_NAME"
RUN mkdir _source
RUN mkdir _build
RUN mkdir _package
RUN mkdir $SOURCE_DIR
RUN mkdir $BUILD_DIR
RUN mkdir $PACKAGE_DIR
RUN mkdir $SOURCE_DIR/webrtc/
WORKDIR $SOURCE_DIR/webrtc/
RUN yes | gclient
RUN fetch --nohooks webrtc
WORKDIR $SOURCE_DIR/webrtc/src
RUN git reset --hard
RUN git clean -xdf
WORKDIR $SOURCE_DIR/webrtc/src/third_party
RUN git reset --hard
RUN git clean -xdf
WORKDIR $SOURCE_DIR/webrtc/src
RUN git fetch
RUN git checkout -f $WEBRTC_COMMIT
WORKDIR $SOURCE_DIR/webrtc
RUN echo "target_os = [ 'android' ]" >> .gclient
RUN gclient sync -v -D
WORKDIR $SOURCE_DIR/webrtc/src
COPY patches/4k.patch /root/patches/
COPY patches/android_webrtc_version.patch /root/patches/
COPY patches/android_fixsegv.patch /root/patches/
COPY patches/android_simulcast.patch /root/patches/
COPY patches/generate_license.patch /root/patches/
COPY patches/android_onremovetrack.patch /root/patches/
COPY android/WebrtcBuildVersion.java $SOURCE_DIR/webrtc/src/sdk/android/api/org/webrtc/
RUN patch -p2 < /root/patches/4k.patch
RUN patch -p1 < /root/patches/android_webrtc_version.patch
RUN patch -p1 < /root/patches/android_fixsegv.patch
RUN patch -p1 < /root/patches/android_simulcast.patch
RUN patch -p1 < /root/patches/generate_license.patch
RUN python $SOURCE_DIR/webrtc/src/tools_webrtc/android/build_aar.py \
    --build-dir $BUILD_DIR/webrtc_android \
    --output $BUILD_DIR//webrtc_android/libwebrtc.aar \
    --arch armeabi-v7a arm64-v8a \
    --extra-gn-args ' \
        is_java_debug=false \
        rtc_include_tests=false \
        rtc_use_h264=false \
        is_component_build=false \
        use_rtti=true \
      '

# armeabi-v7a 用の libwebrtc.a
RUN gn gen $BUILD_DIR/webrtc_armeabi-v7a --args=' \
    target_os="android" \
    target_cpu="arm" \
    is_debug=false \
    rtc_use_h264=false \
    rtc_include_tests=false \
    rtc_build_json=true \
    use_rtti=true \
  '
RUN ninja -C $BUILD_DIR/webrtc_armeabi-v7a
RUN ninja -C $BUILD_DIR/webrtc_armeabi-v7a native_api

WORKDIR $BUILD_DIR/webrtc_armeabi-v7a/obj
RUN $SOURCE_DIR/webrtc/src/third_party/llvm-build/Release+Asserts/bin/llvm-ar -rc $BUILD_DIR/webrtc_armeabi-v7a/libwebrtc.a `find . -name '*.o'`

WORKDIR $SOURCE_DIR/webrtc/src
# arm64-v8a 用の libwebrtc.a
RUN gn gen $BUILD_DIR/webrtc_arm64-v8a --args=' \
    target_os="android" \
    target_cpu="arm64" \
    is_debug=false \
    rtc_use_h264=false \
    rtc_include_tests=false \
    rtc_build_json=true \
    use_rtti=true \
  '
RUN ninja -C $BUILD_DIR/webrtc_arm64-v8a
RUN ninja -C $BUILD_DIR/webrtc_arm64-v8a native_api

WORKDIR $BUILD_DIR/webrtc_arm64-v8a/obj
RUN $SOURCE_DIR/webrtc/src/third_party/llvm-build/Release+Asserts/bin/llvm-ar -rc $BUILD_DIR/webrtc_arm64-v8a/libwebrtc.a `find . -name '*.o'`

# onremovetrack 入りの aar
# ソースが書き換わるので最後にやる必要がある
WORKDIR $SOURCE_DIR/webrtc/src
RUN patch -p1 < /root/patches/android_onremovetrack.patch
RUN python $SOURCE_DIR/webrtc/src/tools_webrtc/android/build_aar.py \
      --build-dir $BUILD_DIR/webrtc_android \
      --output $BUILD_DIR//webrtc_android/libwebrtc_onremovetrack.aar \
      --arch armeabi-v7a arm64-v8a \
      --extra-gn-args ' \
        is_java_debug=false \
        rtc_include_tests=false \
        rtc_use_h264=false \
        is_component_build=false \
        use_rtti=true \
      '
ENV STATIC_DIR "/root/static"
COPY scripts/package_webrtc_android.sh /root/scripts/
COPY VERSION /root/
RUN /root/scripts/package_webrtc_android.sh $STATIC_DIR $SOURCE_DIR $BUILD_DIR $PACKAGE_DIR /root/VERSION
RUN mv $PACKAGE_DIR/webrtc.tar.gz /
FROM ubuntu:18.04
COPY --from=builder /webrtc.tar.gz /