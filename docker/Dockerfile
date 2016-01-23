FROM ubuntu:15.10
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
ENV TERM=linux
ENV DEBIAN_FRONTEND noninteractive
ENV DISPLAY 192.168.10.1:0
RUN apt-get -q update && apt-get -qy install ca-certificates curl
ADD mitmproxy-ca-cert.pem /usr/share/ca-certificates/mitmproxy-ca-cert.pem
RUN echo "mitmproxy-ca-cert.pem" >> /etc/ca-certificates.conf
RUN update-ca-certificates
CMD ["/sbin/init"]
