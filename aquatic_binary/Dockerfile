


FROM debian:stable-slim

RUN mkdir /workspace
WORKDIR /workspace

COPY . /workspace


RUN chmod +x ./env-native-cpu-without-avx-512
RUN ./env-native-cpu-without-avx-512
RUN chmod +x ./aquatic

ENTRYPOINT ["./aquatic"]

