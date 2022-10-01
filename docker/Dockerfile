FROM crcady/grpc:1.49.1-22.04 as base
WORKDIR /tmp/
RUN git clone --depth 1 -b v1.49.1 https://github.com/grpc/grpc
RUN mkdir -p /tmp/grpc/examples/helloworld/cpp/cmake/build
WORKDIR /tmp/grpc/examples/cpp/helloworld/cmake/build
RUN cmake -DCMAKE_PREFIX_PATH=/tmp/grpc_install ../..
RUN make -j
RUN mkdir -p installfiles && cp greeter* *.a ./installfiles/

FROM crcady/grpc:1.49.1-22.04 as final
RUN mkdir -p /app/
COPY --from=base /tmp/grpc/examples/cpp/helloworld/cmake/build/installfiles /app/

WORKDIR /app/
CMD ./greeter_server